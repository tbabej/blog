---
layout: post
title: Securing your user session with Yubikey
---

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

To allow our user to interact with the device, we need to create a privileged
group which will have access to the Yubikey (and also will be restricted to use
one when interacting with the computer).

To do that, create the `yubikey` group and add your user to it.

     sudo groupadd yubikey
     sudo usermod -aG yubikey $USER

Now we can create an udev rule that allows access to the device for all members
of `yubikey`, by creating a `/etc/udev/rules.d/90-yubikey.rules` file containing:

    SUBSYSTEMS=="usb", ATTR{ID_MODEL_FROM_DATABASE}=="Yubico.com Yubikey 4 OTP+U2F+CCID", MODE="0666", GROUP="yubikey"

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

    $ ykpamcfg -2 -v
    debug: util.c:212 (check_firmware_version): YubiKey Firmware version: 4.3.5
    
    Directory /home/tbabej/.yubico created successfully.
    Sending 63 bytes HMAC challenge to slot 2
    Sending 63 bytes HMAC challenge to slot 2
    Stored initial challenge and expected response in '/home/.tbabej/challenge-5675024'.

This created the challenge file named `challenge-5675024`. This challenge will
be used by `pam_yubico` PAM module to cryptographically verify that your
Yubikey is inserted.

Now we're ready to put `pam_yubico` into our PAM stack. Put the following two
lines on top of `/etc/pam.d/login`, `/etc/pam.d/system-auth` and
`/etc/pam.d/password-auth`:

    auth [success=1 default=ignore] pam_succeed_if.so quiet user notingroup yubikey
    auth        requisite    pam_yubico.so mode=challenge-response debug

This two directives make the successful challenge-response for users in the
`yubikey` group mandatory during user login authentication (first time logging,
logging into the machine using ssh, unlocking the screen saver).

Before logging out, make sure you can still log in as a privileged user that is
not member of the yubikey group, like root, using a password.

#### Testing it out

To test it out, try to login as your user, ideally in a different terminal
console. You should be able to login if and only if the yubikey is in one of
the USB slots.

The same applies for your screen saver.

If your login is failing and you're running SELinux, see below.

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

### Locking your session when Yubikey is absent

The second mechanism is to ensure that your computer does not allow you to use
it unless Yubikey is plugged in. We ensure that by setting up an udev rule that
executes a script to lock the screen when Yubikey is removed.

To allow an external script (executed by system service, running under root) to
lock our session, we first create a one-shot systemd service for screen locking
which is going to run under the credentials and session of our user instead.

Put the following unit file into `~/.config/systemd/user/screenlock.service`:

    [Unit]
    Description=Run screenlock
    After=graphical.target
    
    [Service]
    Type=simple
    Environment=DISPLAY=:0
    ExecStart=/usr/bin/i3lock
    
    [Install]
    WantedBy=multi-user.target

Running `systemctl --user start screenlock.service` should now lock your screen
(which you can only unlock if Yubikey is present).

The following script will take care of running the service from within the root
session. Please replace the `<UID>` and `<command to lock your screen>` with
the ones relevant to your setup:

    if [ -z "$(pidof Xorg)" ]
    then
      /usr/bin/systemd-cat -t "yubikey-screen_lock-trigger" /usr/bin/echo "YUBIKEY REMOVED - LOCK NOT ACTIVATED (X session absent)"
    elif [ -z "$(lsusb | grep Yubikey)" ]
    then
      /usr/bin/systemd-cat -t "yubikey-screen_lock-trigger" /usr/bin/echo "YUBIKEY REMOVED - SCREEN LOCK ACTIVATED"
      su -c 'XDG_RUNTIME_DIR="/run/user/<UID>" DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/<UID>/bus" systemctl --user start screenlock' <USERNAME>
    fi

Place the result (with replaced parts) to a proper place, i.e.
/usr/local/bin/yubikey-lock.sh and make it executable using

    $ sudo chmod +x /usr/local/bin/yubikey-lock.sh

At this point you can check the functionality of the script by removing the
Yubikey and running the `yubikey-lock.sh` manually from a root shell. It should
lock your screen and prevent you logging without replugging the Yubikey.

    $ su
    # yubikey-lock.sh

Now we need to setup the udev rule that will make sure that this script is
executed once Yubikey removal is detected. Create a file
`/etc/udev/rules.d/90-yubikey-lock.rules` and insert the following rule inside:

    KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", ATTRS{idProduct}=="0113|0114|0115|0116|0120|0402|0403|0406|0407|0410",
    TAG+="uaccess", GOTO="yubikey_end" ACTION=="remove",
    RUN+="/usr/local/bin/yubikey-lock.sh" LABEL="yubikey_end"

To make sure rule applies, use:

    sudo udevadm control --reload-rules

Now happily remove your Yubikey and falsly feel more secure (remember kids,
with physical access it's game over anyway). However, you can still relish the
feeling, it is a bit reassuring to see the screen go blank immediatelly after
pulling out your smartcard device.

If you are encountering permission errors and access denialis, you might need
to follow the same steps for the SELinux policy update as we did in the
previous section.

#### Bonus part: How not to forget your Yubikey plugged in when leaving the computer

Use a retractable locking reel, attach your Yubikey to it. When leaving, you'll
have to unplug.  You won't forget to lock your screen ever.

Congratulations - your Yubikey now truly became a key for your computer.

Sources:
* <https://vtluug.org/wiki/Yubikey#i3lock_2>
* <https://www.reddit.com/r/i3wm/comments/85ics6/how_to_use_a_password_and_yubikey_with_i3lock://www.reddit.com/r/i3wm/comments/85ics6/how_to_use_a_password_and_yubikey_with_i3lock/>
* <https://wiki.archlinux.org/index.php/yubikey>
