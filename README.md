# 2020-openvpn-mfa-google-auth

Summary: this readme describes how to set up OpenVPN on Ubuntu 18.04
to use MFA using Google Authenticator and PAM.

Each OpenVPN user is linked to a Unix system user. New users are given
the usual OpenVPN files (config file and certificate), plus a QR code
or URL that they can visit to set up MFA on their device.

In the end, users will log on with their username and use an MFA token
as their password.


## Table of Contents

* [2020\-openvpn\-mfa\-google\-auth](#2020-openvpn-mfa-google-auth)
    * [Step 1: Install OpenVPN](#step-1-install-openvpn)
    * [Step 2: Install Dependencies](#step-2-install-dependencies)
    * [Step 3: Create a Google Auth User/Group](#step-3-create-a-google-auth-usergroup)
    * [Step 4: Add PAM to OpenVPN Server Config File](#step-4-add-pam-to-openvpn-server-config-file)
    * [Step 5: PAM Configuration](#step-5-pam-configuration)
    * [Step 6: Update Client Config File](#step-6-update-client-config-file)
    * [Step 7: Register Users](#step-7-register-users)


## Step 1: Install OpenVPN

Skipping this step, will refer to [Digital Ocean's
guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-18-04)
to installing OpenVPN on Ubuntu 18.04.


## Step 2: Install Dependencies

Install two dependencies:

* libpam-google-authenticator - provides a way to connect Google Authenticator app
  to PAM (an auth system that is built into Unix)
* libqrencode3 - package to handle generating QR codes

```
apt install libqrencode3 libpam-google-authenticator
```


## Step 3: Create a Google Auth User/Group

Create a user and group named `gauth` on the Unix system. This is the user and group
used to run the Google Authenticator binary to generate tokens.

As root:

```
addgroup gauth
useradd -g gauth gauth
mkdir /etc/openvpn/google-authenticator
chown gauth:gauth /etc/openvpn/google-authenticator
chmod 0700 /etc/openvpn/google-authenticator
```


## Step 4: Add PAM to OpenVPN Server Config File

Edit the VPN configuration file. We will use `/etc/openvpn/server.conf`
for this example.

Add this line to `/etc/openvpn/server.conf`:

```
plugin /usr/lib/x86_64-linux-gnu/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn
```

**NOTE:** If you change `openvpn` to `login`, OpenVPN will use the Unix
system users and passwords as the VPN users and passwords, with no MFA.
The last word in the line is an "action" that corresponds to a file with
that name in `/etc/pam.d/`.


## Step 5: PAM Configuration

In the prior step we told OpenVPN to use the PAM plugin, and to look for
an "action" called `openvpn`, which will be in the file `/etc/pam.d/openvpn`.

Create the file `/etc/pam.d/openvpn` with the following contents:

```
auth required /lib/x86_64-linux-gnu/security/pam_google_authenticator.so secret=/etc/openvpn/google-authenticator/${USER} user=gauth forward_pass
```

Note that this assumes Google Authenticator user secrets will be located in 
`/etc/openvpn/google-authenticator/`. This can be changed but requires changes
in the steps that follow too.

Note that users are ONLY required to enter the MFA token, not a password. This may seem less secure.
However, the user must also have the corresponding client key file, so it is basically the same as 2FA
plus passwordless key access.


## Step 6: Update Client Config File

The following lines need to be added to the client configuration file to ensure
the client uses the server certificate and performs authentication correctly:

```
remote-cert-tls server
auth-user-pass
```

## Step 7: Register Users

**Unix User Step:**

Issue this command to create a new user:

```
USERNAME = "<username>"
useradd -s /bin/nologin "${USERNAME}"
echo "${USERNAME}:$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 13)" | chpasswd
```

**MFA Registration Step:**

The following assumes your MFA directory (where files generated by MFA registration setup step go) is set like so:

```
MFA_DIR="/etc/openvpn/google-authenticator"
```

This command will create a QR code and a URL for users to sign up for MFA with their device:

This command 

```
MFA_LABEL="My VPN"
sudo -H -u gauth google-authenticator -t -w3 -e10 -d -r3 -R30 -f -l "${MFA_LABEL}" -s ${MFA_DIR}/${USERNAME} > ${MFA_DIR}/${USERNAME}.log
```

Now extract the Authenticator ID that was just generated and use it to generate a PNG image of a QR code:

```
AUTH_ID="$( head -n1 /etc/openvpn/google-authenticator/${USERNAME} )"
qrencode -o /etc/openvpn/google-authenticator/${USERNAME}_qr.png -d 300 -s 10 "otpauth://totp/${USERNAME}?secret=${AUTH_ID}&issuer=${!MFA_LABEL}"
```

Extract the URL for the Authenticator signup and put it in a text file:

```
cat /etc/openvpn/google-authenticator/${USERNAME} | awk '{if(NR==2) print $0}' > /etc/openvpn/google-authenticator/${USERNAME}_mfa_url.txt
```

And finally, a text file with the users' backup codes:

```
tail -n10 /etc/openvpn/google-authenticator/${USERNAME} > /etc/openvpn/google-authenticator/${USERNAME}_backup_codes.txt
```

