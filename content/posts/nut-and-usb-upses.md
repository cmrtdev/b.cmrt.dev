+++
date = '2025-08-31T13:30:44+02:00'
draft = false
title = 'NUT and USB UPSes'
+++
I just migrated my homelab to a new machine, and in the process I had to reinstall Network UPS Tools (more commonly known as [NUT](https://networkupstools.org/)).

In the process, I relearned why you need to make sure NUT is allowed to access USB devices, and how to do that.

NUT is configured as a `netserver` to provide power information to all the machines that are connected to it. I backed up my configuration from my old server so, other than making a couple configuration tweaks, this should've been a piece of cake...

## Thank you, udev

To make a relatively long story short: my UPS tells me it is an "APC Back-UPS RS 550G", whose ID in `/etc/nut/ups.conf` I configured as `apcbackups`[^confusing].

NUT uses drivers[^whatsadriver] to talk to UPSes and, since I'm on Debian, systemd auto-creates a service unit[^notreallyunits] for each UPS you configure and calls it `ups-driver@{ups-name}.service`; in my case, since the `ups.conf` entry for the UPS looks like this

```ini
[apcbackups]
  driver = "usbhid-ups"
  port = "auto"
  desc = "APC Back-UPS RS 550G"
  vendorid = "051D"
  productid = "0002"
```

the unit is `nut-driver@apcbackups.service`, and its logs give us a hint at what the problem is:

```console
root@Server:~# journalctl -xeu nut-driver@apcbackups.service
[...]
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: Network UPS Tools - Generic HID driver 0.52 (2.8.1)
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: USB communication driver (libusb 1.0) 0.46
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: libusb1: Could not open any HID devices: insufficient permissions on everything
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: No matching HID UPS found
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: upsnotify: notify about state 4 with libsystemd: was requested, but not running as a service unit now, will not spam more about it
Aug 31 13:17:11 Server nut-driver@apcbackups[634743]: upsnotify: failed to notify about state 4: no notification tech defined, will not spam more about it
Aug 31 13:17:11 Server nut-driver@apcbackups[634605]: Driver failed to start (exit status=1)
Aug 31 13:17:11 Server nut-driver@apcbackups[634605]: Network UPS Tools - UPS driver controller 2.8.1
Aug 31 13:17:11 Server systemd[1]: nut-driver@apcbackups.service: Control process exited, code=exited, status=1/FAILURE
```

It turns out I (well, `nut`) have **`insufficient permissions on everything`**.

What I was missing was the udev configuration that lets `nut-server` access the USB device. With many thanks to the [Alpine Linux Wiki](https://wiki.alpinelinux.org/wiki/Nut-ups), I created `/etc/udev/rules.d/62-nut-usbups.rules`

```console
# cat /etc/udev/rules.d/62-nut-usbups.rules 
ATTR{idVendor}=="051d", ATTR{idProduct}=="0002", MODE="664", GROUP="nut"
```

Note that the two `ATTR` match the `idVendor` and `idProduct` that I see using `lsusb`:

```console
# lsusb -v
[...]
Bus 003 Device 004: ID 051d:0002 American Power Conversion Uninterruptible Power Supply
Negotiated speed: Full Speed (12Mbps)
Device Descriptor:
[...]
  idVendor           0x051d American Power Conversion
  idProduct          0x0002 Uninterruptible Power Supply
[...]
```

I reloaded udev rules (as instructed)

```bash
udevadm control --reload-rules
```

then  **unplugged and plugged the UPS back in** (this is important[^haveyoutried]).

I got the new bus/device combo (it may or may not change after a reconnection)

```console
# lsusb
Bus 003 Device 004: ID 051d:0002 American Power Conversion Uninterruptible Power Supply
```

checked that the `nut` user had privileges on the device

```console
# ls -l /dev/bus/usb/003/004
crw-rw-r-- 1 root nut 189, 259 Aug 31 13:50 /dev/bus/usb/003/004
```

which showed me that the `nut` group had `rw` permissions on the device, so I restarted `nut-driver@apcbackups.service` and `nut-server.service`

```bash
systemctl restart nut-driver@apcbackups.service
systemctl restart nut-server.service
```

and since neither command returned any error I checked that the UPS was now detected and working

```console
# upsc -L 127.0.0.1
Init SSL without certificate database
apcbackups: APC Back-UPS RS 550G

# upsc apcbackups@127.0.0.1
Init SSL without certificate database
battery.charge: 100
battery.charge.low: 10
battery.charge.warning: 50
[...]
```

and it was :)

## Bonus feature: TrueNAS UPS reporting

If you run TrueNAS with `nut` configured as `netserver` on another machine, and your charts are always blank, this is the workaround I keep having to look up: <https://ixsystems.atlassian.net/browse/NAS-132924?focusedCommentId=289352>.

This workaround does not survive upgrades, so the pre-init script is kind of a must.

I made a slight change to the instructions: I keep both files in `/root/nut-fix`

`/root/nut-fix/nut_fix.conf`

```bash
# Configure remote UPS address:port here...
MY_NUT_HOST="192.168.X.X:3493"

nut_get_all() {
  run -t $nut_timeout upsc -l ${MY_NUT_HOST} || echo "skip-get-values"
}

nut_get() {
  if [ $1 == "skip-get-values" ]; then
    return 0;
  fi

  run -t $nut_timeout upsc "${1}@${MY_NUT_HOST}"

  if [ "${nut_clients_chart}" -eq "1" ]; then
    printf "ups.connected_clients: "
    run -t $nut_timeout upsc -c "${1}@${MY_NUT_HOST}" | wc -l
  fi
}
```

`/root/nut-fix/post_upgrade`

```bash
#!/bin/bash
#
# Run after an OS upgrade
#

# Fix netdata SLAVE UPS bug
if [ -f /usr/lib/netdata/charts.d/nut_ups.chart.sh \
     -a -d /etc/netdata/charts.d \
     -a ! -f /etc/netdata/charts.d/nut_ups.conf ]; then
  cp /root/bin/nut_ups.conf /etc/netdata/charts.d/
  pgrep netdata >/dev/null && systemctl restart netdata || true
fi
```

I ran it (after a `chmod +x /root/nut-fix/post_upgrade`), which confirmed that it worked.

Then in _TrueNAS > System > Advanced Settings > Init/Shutdown Scripts_ I added a

* Type: Script
* Script: `/root/nut-fix/post_upgrade`
* When: Pre Init
* Timeout: 10

To make TrueNAS check that the fix is in place every time it boots (which includes after an upgrade that deleted it).

[^whatsadriver]: <https://networkupstools.org/docs/user-manual.chunked/Overview.html#_drivers>
[^confusing]: Yes, it does sound like "APC Backups". I don't care.
[^haveyoutried]: As you may have guessed, at first it wasn't working and it was because I'd forgotten to do this...
[^notreallyunits]: _Technically_ this is not a unit, but what systemd calls an "instantiated" service: the "service template" is `nut-driver.service` and the thing between the `@` and `.service` is a parameter. Read [this manpage](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Service%20Templates) for more.
