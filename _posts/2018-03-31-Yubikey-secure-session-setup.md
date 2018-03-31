---
layout: post
title: Securing your user session with Yubikey
---

First install this library:

    sudo dnf install pam_yubico

Put the following two lines on top of `/etc/pam.d/login`.

    auth [success=1 default=ignore] pam_succeed_if.so quiet user notingroup yubikey
    auth		required	pam_yubico.so mode=challenge-response

This two directives make the successful challenge-response for users in the `yubikey` group mandatory.

Next, you need setup a slot on your Yubikey to support Challenge-Response protocol. Usually this is done on the second slot, but can be also setup on the first one.

    sudo ykpersonalize -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible

Now, 

Sources:
https://vtluug.org/wiki/Yubikey#i3lock_2
