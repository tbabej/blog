---
layout: post
title: Securing your user session with Yubikey
---

## Yubikey session protection setup

In this post, we will describe how to setup your Yubikey as session-required
smartcard - your user session will work if your Yubikey is attached to the
computer.

We achieve this with following two mechanisms:

* On user login, Yubikey's presence in the USB slot is required
* When Yubikey is not present during X session, screen lock is issued

### Yubikey for login (PAM stack)

The first use case to address is to require Yubikey in order to successfully
login into the machine.

We need to setup basic Yubico tools, if we do not have them:

    curl -sO https://developers.yubico.com/yubico-c/Releases/libyubikey-1.13.tar.gz.sig
    curl -sO https://developers.yubico.com/yubico-c/Releases/libyubikey-1.13.tar.gz
    curl -sO https://developers.yubico.com/yubikey-personalization/Releases/ykpers-1.17.3.tar.gz.sig
    curl -sO https://developers.yubico.com/yubikey-personalization/Releases/ykpers-1.17.3.tar.gz

Verify signatures of both of the libraries, unpack and install them.

Additionally, install PAM module for yubico:

    sudo dnf install pam_yubico

Next, you need setup a slot on your Yubikey to support Challenge-Response
protocol. Usually this is done on the second slot, but can be also setup on the
first one.

    $ sudo ykpersonalize -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible
    Configuration data to be written to key configuration 2:
    
    fixed: m:
    uid: n/a
    key: h:<key hash>
    acc_code: h:000000000000
    OATH IMF: h:0
    ticket_flags: CHAL_RESP
    config_flags: CHAL_HMAC|HMAC_LT64
    extended_flags: SERIAL_API_VISIBLE

This command just configures the second slot of Yubikey. Now, we are ready to
create a challenge file which will be used to authorize this particular Yubikey
during login.

    $ sudo ykpamcfg -2 -v
    debug: util.c:212 (check_firmware_version): YubiKey Firmware version: 4.3.5
    
    Directory /root/.yubico created successfully.
    Sending 63 bytes HMAC challenge to slot 2
    Sending 63 bytes HMAC challenge to slot 2
    Stored initial challenge and expected response in '/root/.yubico/challenge-5675024'.

This created the challenge file named `challenge-5675024`. Now, `pam_yubico`
(which we will setup in a second) expects challenge file name of the form
`<username>-number`, so in  case we need to rename the file:

    # cd /root/.yubico
    # mv challenge-5675024 tbabej-5675024

Now we're ready to put `pam_yubico` into our PAM stack. Put the following two
lines on top of `/etc/pam.d/login`.

    auth [success=1 default=ignore] pam_succeed_if.so quiet user notingroup yubikey
    auth		required	pam_yubico.so mode=challenge-response /root/.yubico/challenge-5675024

This two directives make the successful challenge-response for users in the
`yubikey` group mandatory during user login authentication (first time logging,
logging into the machine using ssh, but not unlocking the screen saver).

At this point, none of the rules are enforced, since your current user isn't
member of the yubikey group. Before doing so, make sure you can login to your
root account using a password, because you can lock yourself out of your usual
user account in case of misconfiguration.

If you're ready, create the `yubikey` group and add your user to it.

     sudo groupadd yubikey
     sudo usermod -aG yubikey $USER

#### Testing it out

To test it out, try to login as your user, ideally in a different terminal
console. You should be able to login if and only if the yubikey is in one of
the USB slots.

If your login is failing and your running SELinux, see below.

#### SELinux policy (Fedora, CentOS)

This subsection is important if you're running a SELinux-powered distro.
Temporarily disable the SELinux protection by going into permissive mode.

    # setenforce 0

Now attempt to login. If the fix helped, the SELinux was indeed an issue, and
you can create a SELinux policy allowing access to necessary resources using
the following:

    # grep avc /var/log/audit/audit.log | audit2allow -M yubikey
    # checkmodule -M -m -o yubikey.mod yubikey.te
    # semodule_package -o yubikey.pp -m yubikey.mod
    # semodule -i yubikey.pp

Sources:
https://vtluug.org/wiki/Yubikey#i3lock_2
