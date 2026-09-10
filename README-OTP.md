# OpenVPN RADIUS + TOTP / Google Authenticator

This setup adds TOTP-based two-factor authentication to OpenVPN while keeping
the existing RADIUS username/password authentication.

The OpenVPN client displays three separate fields:

- Username
- Password
- Verification code

The user does not need to append the OTP code to the password.

Authentication succeeds only when both checks pass:

1. Username/password authentication against RADIUS
2. TOTP authentication through PAM / Google Authenticator

## Authentication flow

OpenVPN uses `static-challenge` to collect the OTP separately from the password.

The client sends the password and OTP to OpenVPN in SCRV1 format:

    SCRV1:<base64-password>:<base64-response>

The modified RADIUS plugin detects SCRV1 and extracts the original RADIUS
password before sending the authentication request to the RADIUS server.

The standard OpenVPN PAM authentication plugin independently reads the same
SCRV1 structure and passes the OTP response to the Google Authenticator PAM
module.

Authentication flow:

    OpenVPN Client

      Username
      Password
      Verification code
            |
            v
    OpenVPN Server
            |
            +-----------------------------+
            |                             |
            v                             v
    radiusplugin-otp.so          openvpn-plugin-auth-pam.so
            |                             |
            | SCRV1 decode                | SCRV1 decode
            |                             |
            v                             v
    RADIUS password auth          PAM Google Authenticator
            |                             |
            +-------------+---------------+
                          |
                          v
                  Authentication OK

Both authentication mechanisms must succeed.

## Requirements

Example environment:

    OpenVPN 2.7.x
    FreeRADIUS 3.x
    libpam-google-authenticator
    openvpn-plugin-auth-pam.so

Install Google Authenticator PAM support:

    apt install libpam-google-authenticator

The OpenVPN PAM plugin is normally installed with the OpenVPN package.

Typical location on Ubuntu/Debian:

    /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so

## SCRV1 support in the RADIUS plugin

Without `static-challenge`, OpenVPN normally provides the password as:

    password=MyPassword

With `static-challenge`, OpenVPN provides the credentials as:

    password=SCRV1:<base64-password>:<base64-response>

The original RADIUS plugin sends the value of the OpenVPN `password`
environment variable directly to RADIUS.

This causes RADIUS authentication to fail because the RADIUS server receives
the complete SCRV1 string instead of the user's original password.

The modified plugin detects the `SCRV1:` prefix, decodes the first Base64
field and sends only the original password to RADIUS.

Conceptually:

    if (password starts with "SCRV1:")
    {
        decode password;
        decode OTP;

        newuser->setPassword(decoded_password);
    }
    else
    {
        newuser->setPassword(password);
    }

The OTP value is not validated by the RADIUS plugin.

OTP validation is performed independently by the OpenVPN PAM plugin.

## Build the OTP-enabled RADIUS plugin

Build from source:

    cd /usr/local/src/openvpn-auth-radius

    make clean
    make

Install:

    install -m 755 radiusplugin.so \
      /usr/local/lib/openvpn/radiusplugin-otp.so

Verify:

    ls -l /usr/local/lib/openvpn/radiusplugin-otp.so

## OpenVPN server configuration

Add the following configuration to the OpenVPN instance where OTP should be
enabled:

    static-challenge "Verification code" 0

    plugin /usr/local/lib/openvpn/radiusplugin-otp.so /etc/openvpn/radiusplugin-tcp.cnf

    plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so "openvpn-otp login USERNAME Verification OTP"

    username-as-common-name

Only one RADIUS plugin should be loaded for this OpenVPN instance.

Do not load both the standard and the OTP-enabled RADIUS plugin at the same
time.

## PAM configuration

Create:

    /etc/pam.d/openvpn-otp

with:

    auth required pam_google_authenticator.so secret=/etc/openvpn/otp-secrets/${USER} user=root
    account required pam_permit.so

Do not add:

    auth required pam_unix.so

unless local Linux password authentication is intentionally required.

In this configuration:

    Password authentication -> RADIUS
    OTP authentication      -> PAM / Google Authenticator

## OTP secret storage

Create a central directory for OTP secrets:

    mkdir -p /etc/openvpn/otp-secrets
    chown root:root /etc/openvpn/otp-secrets
    chmod 700 /etc/openvpn/otp-secrets

Each RADIUS username must have a corresponding OTP secret file:

    /etc/openvpn/otp-secrets/<USERNAME>

The filename must match the OpenVPN/RADIUS username exactly.

## Create OTP credentials for a user

Set the username:

    USERNAME="radius_username"

Generate the Google Authenticator secret:

    google-authenticator \
      -t \
      -f \
      -r 3 \
      -R 30 \
      -w 3 \
      -s "/etc/openvpn/otp-secrets/${USERNAME}"

Set secure permissions:

    chown root:root "/etc/openvpn/otp-secrets/${USERNAME}"
    chmod 600 "/etc/openvpn/otp-secrets/${USERNAME}"

The generated QR code can then be scanned using a compatible authenticator
application.

## Verify OTP authentication through PAM

Before testing OpenVPN, verify the OTP configuration directly through PAM:

    pamtester openvpn-otp USERNAME authenticate

Expected prompt:

    Verification code:

Successful authentication:

    pamtester: successfully authenticated

If PAM also asks for a normal password, check `/etc/pam.d/openvpn-otp` for
additional password-based PAM modules such as `pam_unix.so`.

## OpenVPN PAM prompt mapping

The Google Authenticator PAM module produces prompts similar to:

    login:
    Verification code:

The OpenVPN PAM plugin maps these prompts as follows:

    login        -> USERNAME
    Verification -> OTP

Therefore the plugin configuration must include:

    plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so "openvpn-otp login USERNAME Verification OTP"

During troubleshooting, OpenVPN may log messages similar to:

    PLUGIN AUTH-PAM: BACKGROUND: parsed static challenge password
    PLUGIN AUTH-PAM: BACKGROUND: my_conv[0] query='login:' style=2
    PLUGIN AUTH-PAM: BACKGROUND: name match found, query/match-string ['login:', 'login'] = 'USERNAME'
    PLUGIN AUTH-PAM: BACKGROUND: my_conv[0] query='Verification code: ' style=1
    PLUGIN AUTH-PAM: BACKGROUND: name match found, query/match-string ['Verification code: ', 'Verification'] = 'OTP'

## Restart OpenVPN

Restart the OTP-enabled OpenVPN instance:

    systemctl restart openvpn-server@server

Check status:

    systemctl status openvpn-server@server

Follow the OpenVPN log:

    tail -f /var/log/openvpn/server.log

## Authentication behaviour

Authentication succeeds only when both mechanisms return success.

Correct password + correct OTP:

    RADIUS = OK
    PAM OTP = OK
    Result = ALLOW

Correct password + wrong OTP:

    RADIUS = OK
    PAM OTP = FAIL
    Result = DENY

Wrong password + correct OTP:

    RADIUS = FAIL
    PAM OTP = OK
    Result = DENY

Wrong password + wrong OTP:

    RADIUS = FAIL
    PAM OTP = FAIL
    Result = DENY

## Multiple OpenVPN instances

OTP can be enabled only for selected OpenVPN instances.

Example:

    TCP OpenVPN instance -> RADIUS + OTP
    UDP OpenVPN instance -> RADIUS only

Only the OTP-enabled instance needs:

    static-challenge "Verification code" 0

    plugin /usr/local/lib/openvpn/radiusplugin-otp.so /etc/openvpn/radiusplugin-tcp.cnf

    plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so "openvpn-otp login USERNAME Verification OTP"

Other OpenVPN instances can continue using their existing authentication
configuration unchanged.

## Troubleshooting

### RADIUS works without static challenge but fails with static challenge

Verify that the OTP-enabled RADIUS plugin is loaded:

    plugin /usr/local/lib/openvpn/radiusplugin-otp.so /etc/openvpn/radiusplugin-tcp.cnf

The plugin must support SCRV1 decoding:

    SCRV1:<base64-password>:<base64-response>

The decoded original password must be sent to RADIUS instead of the complete
SCRV1 value.

### PAM authentication fails

Verify PAM independently:

    pamtester openvpn-otp USERNAME authenticate

If this fails, troubleshoot PAM before OpenVPN.

### Check OTP secret permissions

    ls -ld /etc/openvpn/otp-secrets
    ls -l /etc/openvpn/otp-secrets/USERNAME

Recommended permissions:

    /etc/openvpn/otp-secrets           root:root 700
    /etc/openvpn/otp-secrets/USERNAME  root:root 600

### Check system time

TOTP authentication depends on accurate system time.

Check:

    timedatectl

Recommended status:

    System clock synchronized: yes
    NTP service: active

### Increase OpenVPN logging temporarily

For troubleshooting:

    verb 7

After troubleshooting, return to a normal production verbosity level:

    verb 3

## Security recommendations

Do not log:

- User passwords
- Decoded passwords
- OTP values
- Google Authenticator secrets

Temporary diagnostic logs that only display credential lengths should also be
removed from production builds.

Protect the OTP secret directory:

    chmod 700 /etc/openvpn/otp-secrets

Protect individual OTP secret files:

    chmod 600 /etc/openvpn/otp-secrets/USERNAME

OTP secret files should not be readable by unprivileged users.

## Production configuration summary

OpenVPN:

    static-challenge "Verification code" 0

    plugin /usr/local/lib/openvpn/radiusplugin-otp.so /etc/openvpn/radiusplugin-tcp.cnf

    plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so "openvpn-otp login USERNAME Verification OTP"

    username-as-common-name

PAM:

    auth required pam_google_authenticator.so secret=/etc/openvpn/otp-secrets/${USER} user=root
    account required pam_permit.so

OTP secret:

    /etc/openvpn/otp-secrets/<USERNAME>

Final authentication flow:

    Username --------------------------+
                                       |
    Password -> SCRV1 decode -> RADIUS +----> OpenVPN access
                                       |
    OTP ------> PAM Google Authenticator+

Access is granted only when both RADIUS password authentication and TOTP
authentication succeed.
