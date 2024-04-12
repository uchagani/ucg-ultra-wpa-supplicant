# Unifi Cloud Gateway Ultra wpa_supplicant bypass ATT fiber modem
All credit for these instructions goes to https://github.com/evie-lau/uxg-lite-wpa-supplicant.  All I'm doing here is updating instructions to work for Unifi Cloud Gateway Ultra.


Use this guide to setup wpa_supplicant with your Unifi Cloud Gateway Ultra or Unifi OS gateway/router to bypass the ATT modem.

This should also work on other Unifi OS devices, just may have to replace the interface name with the device's WAN port interface.

> NOTE: Take note of your Unifi router's WAN port interface name. In the rest of the guide, I'll be using `eth4` because that is the WAN interface for the Unifi Cloud Gateway Ultra. If using another device, replace the interface name appropriately.

Prerequisites:
- extracted and decoded certificates from an ATT modem

Instructions to [extract certs for newish BGW210](https://github.com/mozzarellathicc/attcerts)

## Table of Contents
- [Install wpa_supplicant](#install-wpa_supplicant-on-the-ucg-ultra) - install wpasupplicant on your Unifi gateway
- [Copy certs and config](#copy-certs-and-config-to-ucg-ultra) - copy files generated from mfg_dat_decode tool into Unifi gateway
- [Spoof MAC Address](#spoof-mac-address) - spoof Unifi WAN port to match original ATT gateway MAC address
- [Setup network](#setup-network) - set required network settings (VLAN0) in Unifi dashboard
- [Test wpa_supplicant](#test-wpa_supplicant) - test wpasupplicant
- [Setup wpa_supplicant service for startup](#setup-wpa_supplicant-service-for-startup) - start wpasupplicant on Unifi router bootup
- [Survive firmware updates](#survive-firmware-updates) - automatically restore and setup wpasupplicant after firmware updates wipe it

## Install wpa_supplicant on the Unifi Cloud Gateway Ultra
SSH into your Unifi Cloud Gateway Ultra.

> Unlike all my other Unifi devices, my SSH private key didn't work with my username, but worked with the `root` user instead. Or user + password defined in `Settings` -> `System` -> `Advanced` -> `Device Authentication`.

Unifi OS is a Debian-based distro, so we can install the `wpasupplicant` package.
```
> apt update -y
> apt install -y wpasupplicant
```

Create a `certs` folder in the `/etc/wpa_supplicant` folder.
```
> mkdir -p /etc/wpa_supplicant/certs
```

We'll copy files into here in the next step.

## Copy certs and config to Unifi Cloud Gateway Ultra
Back on your computer, prepare your files to copy into the Unifi Cloud Gateway Ultra.

These files come from the mfg_dat_decode tool:
- CA_XXXXXX-XXXXXXXXXXXXXX.pem
- Client_XXXXXX-XXXXXXXXXXXXXX.pem
- PrivateKey_PKCS1_XXXXXX-XXXXXXXXXXXXXX.pem
- wpa_supplicant.conf

```
> scp *.pem <ip-address-of-ucg>:/etc/wpa_supplicant/certs
> scp wpa_supplicant.conf <ip-address-of-ucg>:/etc/wpa_supplicant
```

Make sure in the `wpa_supplicant.conf` to modify the `ca_cert`, `client_cert` and `private_key` to use absolute paths. In this case, prepend `/etc/wpa_supplicant/certs/` to the filename strings. It should look like the following...
```
...
network={
        ca_cert="/etc/wpa_supplicant/certs/CA_XXXXXX-XXXXXXXXXXXXXX.pem"
        client_cert="/etc/wpa_supplicant/certs/Client_XXXXXX-XXXXXXXXXXXXXX.pem"
        ...
        private_key="/etc/wpa_supplicant/certs/PrivateKey_PKCS1_XXXXXX-XXXXXXXXXXXXXX.pem"
}
```

## Spoof MAC address
We'll need to spoof the MAC address on the WAN port (interface `eth4` on the Unifi Cloud Gateway Ultra) to successfully authenticate with ATT with our certificates.  Spoofing MAC address on the Unifi dashboard works for the Unifi Cloud Gateway Ultra as of Unifi Cloud Gateway Ultra firmware version 3.2.12.  If it does not work follow the instructions below. 

If you're spoofing the  MAC Address via the console, it should look like the below image (Add the MAC address of your AT&T Gateway):

![Alt text](UCGCloneMacAddress.jpg)

I know there's an option in the Unifi dashboard to spoof MAC address on the Internet (WAN) network, but this didn't seem to work when I tested it. (If anyone does test this successfully without needing the following, please let me know).

Instead, I had to manually set it up, based on these [instructions to spoof mac address](https://www.xmodulo.com/spoof-mac-address-network-interface-linux.html).

SSH back into your Unifi Cloud Gateway Ultra, and create the following file.

`vi /etc/network/if-up.d/changemac`

```
#!/bin/sh

if [ "$IFACE" = eth4 ]; then
  ip link set dev "$IFACE" address XX:XX:XX:XX:XX:XX
fi
```
Replace the mac address with your gateway's address, found in the `wpa_supplicant.conf` file.

Set the permissions:
```
> sudo chmod 755 /etc/network/if-up.d/changemac
```
This file will spoof your WAN mac address when `eth4` starts up. Go ahead and run the same command now so you don't have to reboot your Unifi Cloud Gateway Ultra.
```
> ip link set dev "$IFACE" address XX:XX:XX:XX:XX:XX
```

## Setup network

### Set VLAN ID on WAN connection
ATT authenticates using VLAN ID 0, so we have to tag our WAN port with that.

In your Unifi console/dashboard, under `Settings` -> `Internet` -> `Primary (WAN1)` (or your WAN name if you renamed it), Enable `VLAN ID` and set it to `0` and set the `QoS` tag to `1`.

Before applying, note that this change will prevent you from accessing the internet until after running `wpa_supplicant` in the next step. If you need to restore internet access before finishing this setup guide, you can always disable `VLAN ID`.  

![Alt text](vlan0.png)

Apply the change, then unplug the ethernet cable from the ONT port on your ATT Gateway, and plug it into the WAN port on your Unifi Cloud Gateway Ultra.

## Test wpa_supplicant
While SSHed into the Unifi Cloud Gateway Ultra, run this to test the authentication.
```
> wpa_supplicant -i eth4 -D wired -c /etc/wpa_supplicant/wpa_supplicant.conf
```
Breaking down this command...
- `-i eth4` Specifies `eth4` (Unifi Cloud Gateway Ultra WAN port) as the interface
- `-D wired` Specify driver type of `eth4`
- `-c <path-to>/wpa_supplicant.conf` The config file

You should see the message `Successfully initialized wpa_supplicant` if the command and config are configured correctly.

Following that will be some logs from authenticating. If it looks something like this, then it was successful!
```
eth4: CTRL-EVENT-EAP-PEER-CERT depth=0 subject='/C=US/ST=Michigan/L=Southfield/O=ATT Services Inc/OU=OCATS/CN=aut03lsanca.lsanca.sbcglobal.net' hash=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
eth4: CTRL-EVENT-EAP-PEER-ALT depth=0 DNS:aut03lsanca.lsanca.sbcglobal.net
eth4: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
eth4: CTRL-EVENT-CONNECTED - Connection to XX:XX:XX:XX:XX:XX completed [id=0 id_str=]
```
> If you don't see the `EAP authentication completed successfully` message, try checking to make sure the MAC address was spoofed successfully.

`Ctrl-c` to exit. If you would like to run it in the background for temporary internet access, add a `-B` parameter to the command. Running this command is still a manual process to authenticate, and it will only last until the next reboot.

## Setup wpa_supplicant service for startup
Now we have to make sure wpa_supplicant starts automatically when the Unifi Cloud Gateway Ultra reboots.

Let's use wpa_supplicant's built in interface-specific service to enable it on startup. More information [here](https://wiki.archlinux.org/title/Wpa_supplicant#At_boot_.28systemd.29).

Because we need to specify the `wired` driver and `eth4` interface, the corresponding service will be `wpa_supplicant-wired@eth4.service`. This service is tied to a specific .conf file, so we will have to rename our config file.

Back in `/etc/wpa_supplicant`, rename `wpa_supplicant.conf` to `wpa_supplicant-wired-eth4.conf`.
```
> cd /etc/wpa_supplicant
> mv wpa_supplicant.conf wpa_supplicant-wired-eth4.conf
```

Then start the service and check the status.
```
> systemctl start wpa_supplicant-wired@eth4

> systemctl status wpa_supplicant-wired@eth4
```
If the service successfully started and is active, you should see similar logs as when we tested with the `wpa_supplicant` command.

Now we can go ahead and enable the service.
```
> systemctl enable wpa_supplicant-wired@eth4
```

Try restarting your Unifi Cloud Gateway Ultra if you wish, and it should automatically authenticate!

## Survive firmware updates
Firmware updates will nuke the packages installed through `apt` that don't come with the stock Unifi OS, removing our `wpasupplicant` package and service. Since we'll no longer have internet without wpa_supplicant authenticating us with ATT, we can't reinstall it from the debian repos.

Let's cache some files locally and create a system service to automatically reinstall, start, and enable wpa_supplicant again on bootup.

First download the required packages (with missing dependencies) from debian into a persisted folder. These are the resources if you wish to pull the latest download links. Make sure to get the `arm64` package.
- https://packages.debian.org/bullseye/arm64/wpasupplicant/download
- https://packages.debian.org/bullseye/arm64/libpcsclite1/download

```
> mkdir -p /etc/wpa_supplicant/packages
> cd /etc/wpa_supplicant/packages
> wget http://ftp.us.debian.org/debian/pool/main/w/wpa/wpasupplicant_2.9.0-21_arm64.deb
> wget http://ftp.us.debian.org/debian/pool/main/p/pcsc-lite/libpcsclite1_1.9.1-1_arm64.deb
```

> As of the 3.1.15 -> 3.1.16 firmware update, my `/etc/wpa_supplicant` folder did not get wiped, so these should persist through an update for us to reinstall.

Now let's create a service to install these packages and enable/start wpa_supplicant:

```
> vi /etc/systemd/system/reinstall-wpa.service
```

Paste this as the content:
```
[Unit]
Description=Reinstall and start/enable wpa_supplicant
AssertPathExistsGlob=/etc/wpa_supplicant/packages/wpasupplicant*arm64.deb
AssertPathExistsGlob=/etc/wpa_supplicant/packages/libpcsclite1*arm64.deb
ConditionPathExists=!/sbin/wpa_supplicant

[Service]
Type=oneshot
ExecStartPre=dpkg -Ri /etc/wpa_supplicant/packages
ExecStart=systemctl start wpa_supplicant-wired@eth4
ExecStartPost=systemctl enable wpa_supplicant-wired@eth4

[Install]
WantedBy=multi-user.target
```
Now enable the service.
```
> systemctl daemon-reload
> systemctl enable reinstall-wpa.service
```
This service should run on startup. It will check if `/sbin/wpasupplicant` got wiped, and if our package files exist. If both are true, it will install and startup wpa_supplicant.

<details>
<summary><h3>(Optional) If you want to test this...</h3></summary>

```
> systemctl stop wpa_supplicant-wired@eth4
> systemctl disable wpa_supplicant-wired@eth4
> apt remove wpasupplicant -y
```

Now try restarting your Unifi Cloud Gateway Ultra. Upon boot up, SSH back in, and check `systemctl status wpa_supplicant-wired@eth4`.
- Alternatively, without a restart, run `systemctl start reinstall-wpa.service`, wait until it finishes, then `systemctl status wpa_supplicant-wired@eth4`.)

You should see the following:
```
Loaded: loaded (/lib/systemd/system/wpa_supplicant-wired@.service; enabled; vendor preset: enabled)
Active: active (running) ...
...
Dec 29 23:20:00 Unifi Cloud Gateway Ultra wpa_supplicant[6845]: eth4: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
```
</details>

## Troubleshooting

Some problems I ran into...

<a id="fopen"></a>
<details>
  <summary><b>OpenSSL: tls_connection_ca_cert</b></summary>

    > OpenSSL: tls_connection_ca_cert - Failed to load root certificates error:02001002:system library:fopen:No such file or directory

- Make sure in the wpa_supplicant config file to set the absolute path for each certificate, mentioned [here](#copy-certs-and-config-to-ucg-ultra).
</details>

## Additional resources
Special thanks to many of these resources I used to learn all this (nearly from scratch).
- [Guide for USG wpa_supplicant](https://wells.ee/journal/2020-03-01-bypassing-att-fiber-modem-unifi-usg/)
- [ArchWiki wpa_supplicant guide](https://wiki.archlinux.org/title/Wpa_supplicant) where I learned to use wpa_supplicant
- [Spoofing MAC on interfaces](https://www.xmodulo.com/spoof-mac-address-network-interface-linux.html) for spoofing MAC
- [DigitalOcean Systemd unit files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files) for info on systemd unit files
- [Post by /u/superm1](https://www.reddit.com/r/Ubiquiti/comments/18rc0ag/att_modem_bypass_and_unifios_32x_guide/) who posted a similar approach to mine a few days after. I adapted the reinstall service with some extra checks and improvements to also start/enable wpasupplicant after installing.
