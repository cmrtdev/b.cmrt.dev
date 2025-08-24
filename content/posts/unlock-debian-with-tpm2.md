+++
date = '2025-08-22T01:27:04+02:00'
draft = true
title = 'TPM2 auto-unlock of a LUKS-encrypted laptop running Debian'
+++
Today I'm going to try to make my Debian laptop's LUKS encrypted drive auto-unlock with TPM2.

In and out, 20 minute adventure.

## tl;dr

If you're impatient and don't want to know the hell I had to go through:

* Install `dracut` and `tpm2-tools` (the latter is why this post is so long. If it doesn't work, read through, you may be missing some Dracut modules.)
* `echo 'install_optional_items+=" /usr/lib64/libtss2* /usr/lib64/libfido2.so.* "' | sudo tee -a /etc/dracut.conf.d/tss2.conf`
* Add `tpm2-device=auto` in `/etc/crypttab` (on your root partition's line) between `luks` and `discard` (looks like `luks,tpm2-device=auto,discard`)
* Run `dracut -f` and make sure the run doesn't show an error about `tpm2-tss` (or any other error, really)
* Run `systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+7+11+14" /dev/YOUR_DEVICE` (replace `YOUR_DEVICE` with the same, root one as above)
* `reboot`

## Step 1: Dracut

`initramfs-tools` does not support TPM2 unlock; support was supposed to be added - see [this merge request](https://salsa.debian.org/cryptsetup-team/cryptsetup/-/merge_requests/39) from June 2024 - but still hasn't been.

The solution? [`dracut`](https://en.wikipedia.org/wiki/Dracut_(software))[^dracut], of course!

My `root`, LUKS-encrypted device is `/dev/nvme0n1p3`.

After a quick

```bash
sudo apt-get update
sudo apt-get install dracut
```

which also automagically uninstalled `initramfs-tools`, followed by

```bash
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+7+11+14" /dev/nvme0n1p3
```

Which completed correctly[^pcrregisters], I `reboot` and...

_It doesn't work._

## Step 2: /etc/crypttab

Right, I forgot to tell the system that we want to unlock with TPM2.

Let's edit `/etc/crypttab` and change this line (my root device is `/dev/nvme0n1p3`)

```text
nvme0n1p3_crypt UUID=12345678-90ab-cdef-fedc-ba0987654321 none luks,discard
```

to this (adding `tpm2-device=auto`)

```text
nvme0n1p3_crypt UUID=12345678-90ab-cdef-fedc-ba0987654321 none luks,tpm2-device=auto,discard
```

And then do a quick `dracut -f` to regenerate the `initramfs` and...

It _still_ doesn't work.

## Step 3: Recovery

And by "it still doesn't work" what I mean is "it hung while booting, right when it's supposed to ask me for a password if TPM2 unlock fails".

While the earlier reboot actually asked me for a password (it didn't even try the TPM), this one... Just hung :(

In hindsight, I should've paid more attention to the error that showed up when I ran

```text
# dracut -f
dracut[I]: Executing: /usr/bin/dracut -f
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
[...]
dracut[I]: *** Including modules done ***
dracut[I]: *** Installing kernel module dependencies ***
dracut[I]: *** Installing kernel module dependencies done ***
dracut[I]: *** Resolving executable dependencies ***
dracut[I]: *** Resolving executable dependencies done ***
dracut[I]: *** Generating early-microcode cpio image ***
dracut[I]: *** Constructing GenuineIntel.bin ***
dracut[I]: *** Store current command line parameters ***
dracut[I]: *** Stripping files ***
dracut[I]: *** Stripping files done ***
dracut[I]: *** Creating image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' ***
dracut[I]: *** Hardlinking files ***
dracut[I]: *** Hardlinking files done ***
dracut[I]: Using auto-determined compression method 'zstd'
dracut[I]: *** Creating initramfs image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' done ***
dracut[I]: *** Moving image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' to '/boot/initrd.img-6.12.35+deb13-amd64' ***
dracut[I]: *** Moving image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' to '/boot/initrd.img-6.12.35+deb13-amd64' done ***
```

Dracut fails to load the `tpm2-tss` module, which seems kind of important for TPM2 unlock.

```text
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
```

In my defense though: **why did it complete building the `initramfs`?!**

I believe I can be excused for thinking that - at the very least - it was going to ask me for my password if it failed to auto-unlock?

Thankfully in GRUB I had an older kernel (still built by `initramfs-tools`), which booted with no issues and got me back into my system.

I checked what was the most recent kernel I had installed[^eyeball] to figure out which one had a messed up `initramfs`, removed the `tpm2-device=auto` directive from `/etc/crypttab`, then used

```bash
dracut -f --kmod 6.12.35+deb13
```

to regenerate the broken `initramfs`, and my system went back to working (and asking for my LUKS encryption password on boot).

## Step 4: tpm2-tss

Then I tried to install the `tpm2-tss` module that "can't be installed" by `dracut`.

On the Debian Packages site (tracker?) I [searched for `tpm2-tss`](https://packages.debian.org/search?keywords=tpm2-tss). Of all the available packages:

* I ignored `-dev` packages
* and `dbgsym` packages
* which left me with `libengine-tpm2-tss-openssl` and `tpm2-tss-engine-tools`

I installed both, I regenerated the `initramfs`, it gave the same error, but because hope is my strategy here (and I had a quick way of fixing it), I rebooted. It got stuck (obviously).

## Step 5: More debugging

The Debian packages website for `forky` (`testing`, which is the version of Debian I am running) [tells me](https://packages.debian.org/search?searchon=contents&keywords=tpm2-tss&mode=path&suite=testing&arch=any) that the only `tpm2-tss` mention is in `libtss2-doc`, which I don't have.

I installed it, but it (obviously) still didn't work[^duh].

Then I figured out[^rtfm] by inspecting the list of available Dracut modules using `dracut --list-modules`, that our "missing" `tpm2-tss` module is actually available to Dracut:

```text
# dracut --list-modules | grep tpm2
dracut[I]: Executing: /usr/bin/dracut --list-modules
tpm2-tss
```

but that for some reason when I regenerate the `initramfs` it fails to load it. I decided to try and reinstall `dracut`, but it did not help

```text
# apt-get --reinstall install dracut
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
0 upgraded, 0 newly installed, 1 reinstalled, 0 to remove and 711 not upgraded.
Need to get 0 B/14.7 kB of archives.
After this operation, 0 B of additional disk space will be used.
(Reading database ... 257491 files and directories currently installed.)
Preparing to unpack .../archives/dracut_108-3_all.deb ...
Unpacking dracut (108-3) over (108-3) ...
Setting up dracut (108-3) ...
update-initramfs: deferring update (trigger activated)
Processing triggers for man-db (2.13.1-1) ...
Processing triggers for dracut (108-3) ...
update-initramfs: Generating /boot/initrd.img-6.12.35+deb13-amd64
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
```

### Checking Dracut modules

Dracut stores its modules in `/usr/lib/dracut/modules.d`, and the `tpm2-tss` module is has its own `73tpm2-tss` subfolder. It only contains a shell script - `module-setup.sh` - which (I believe) takes care of installing the module into an `initramfs` when Dracut runs.

This script tries to install several files in the `initramfs`:

```sh
# Install the required file(s) and directories for the module in the initramfs.
install() {
    inst_sysusers tpm2-tss.conf

    inst_multiple -o \
        "$tmpfilesdir"/tpm2-tss-fapi.conf \
        "$udevrulesdir"/60-tpm-udev.rules \
        "$systemdutildir"/system-generators/systemd-tpm2-generator \
        "$systemdsystemunitdir/tpm2.target" \
        tpm2_pcrread tpm2_pcrextend tpm2_createprimary tpm2_createpolicy \
        tpm2_create tpm2_load tpm2_unseal tpm2

    # Install library file(s)
    _arch=${DRACUT_ARCH:-$(uname -m)}
    inst_libdir_file \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-esys.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-fapi.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-mu.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-rc.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-sys.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-tcti-cmd.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-tcti-device.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-tcti-mssim.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-tcti-swtpm.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libtss2-tctildr.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libcryptsetup.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"/cryptsetup/libcryptsetup-token-systemd-tpm2.so" \
        {"tls/$_arch/",tls/,"$_arch/",}"libcurl.so.*" \
        {"tls/$_arch/",tls/,"$_arch/",}"libjson-c.so.*"

}
```

So I decided to make sure I had them available in my system:

* `libtss2-esys.so.*` should be provided by [`libtss2-esys-3.0.2-0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-esys)
* `libtss2-fapi.so.*` should be provided by [`libtss2-fapi1t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-fapi)
* `libtss2-mu.so.*` should be provided by [`libtss2-mu-4.0.1-0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-mu)
* `libtss2-rc.so.*` should be provided by [`libtss2-rc0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-rc)
* `libtss2-sys.so.*` should be provided by [`libtss2-sys1t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-sys)
* `libtss2-tcti-cmd.so.*` should be provided by [`libtss2-tcti-cmd0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-tcti-cmd)
* `libtss2-tcti-device.so.*` should be provided by [`libtss2-tcti-device0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-tcti-device)
* `libtss2-tcti-mssim.so.*` should be provided by [`libtss2-tcti-mssim0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-tcti-mssim)
* `libtss2-tcti-swtpm.so.*` should be provided by [`libtss2-tcti-swtpm0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-tcti-swtpm)
* `libtss2-tctildr.so.*` should be provided by [`libtss2-tctildr0t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libtss2-tctildr)
* `libcryptsetup.so.*` should be provided by [`libcryptsetup12`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libcryptsetup.so)
* `/cryptsetup/libcryptsetup-token-systemd-tpm2.so` should be provided by [`systemd-cryptsetup`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libcryptsetup-token-systemd-tpm2.so)
* `libcurl.so.*` should be provided by [`libcurl4t64`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libcurl.so)
* `libjson-c.so.*` should be provided by [`libjson-c5`](https://packages.debian.org/search?suite=forky&arch=any&mode=filename&searchon=contents&keywords=libjson-c.so)

Checking for missing libraries with `apt-cache policy` I find out that I'm missing `libtss2-fapi1t64`

```text
$ apt-cache policy libtss2-esys-3.0.2-0t64 libtss2-fapi1t64 libtss2-mu-4.0.1-0t64 libtss2-rc0t64 libtss2-sys1t64 libtss2-tcti-cmd0t64 libtss2-tcti-device0t64 libtss2-tcti-mssim0t64 libtss2-tcti-swtpm0t64 libtss2-tctildr0t64 libcryptsetup12 systemd-cryptsetup libcurl4t64 libjson-c5 | grep -B 1 Installed
libtss2-esys-3.0.2-0t64:
  Installed: 4.1.3-1.2
--
libtss2-fapi1t64:
  Installed: (none)
--
libtss2-mu-4.0.1-0t64:
  Installed: 4.1.3-1.2
--
libtss2-rc0t64:
  Installed: 4.1.3-1.2
--
libtss2-sys1t64:
  Installed: 4.1.3-1.2
--
libtss2-tcti-cmd0t64:
  Installed: 4.1.3-1.2
--
libtss2-tcti-device0t64:
  Installed: 4.1.3-1.2
--
libtss2-tcti-mssim0t64:
  Installed: 4.1.3-1.2
--
libtss2-tcti-swtpm0t64:
  Installed: 4.1.3-1.2
--
libtss2-tctildr0t64:
  Installed: 4.1.3-1.2
--
libcryptsetup12:
  Installed: 2:2.7.5-2
--
systemd-cryptsetup:
  Installed: 257.7-1
--
libcurl4t64:
  Installed: 8.14.1-2
--
libjson-c5:
  Installed: 0.18+ds-1
```

but even after installing it, `dracut` is still returning the same error.

### libtss2-dev

One thing I noticed on Debian Packages about the many `libtss2-*` libraries above is that they also appear in `libtss2-dev`, which I don't have

```text
# apt-cache policy libtss2-dev
libtss2-dev:
  Installed: (none)
  Candidate: 4.1.3-1.2
  Version table:
     4.1.3-1.2 500
        500 http://deb.debian.org/debian testing/main amd64 Packages
```

While in theory I shouldn't need it (it's a `-dev` library, not a runtime one), maybe installing it will also pull in missing dependencies that I don't know I need?

```text
# apt-get install libtss2-dev
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  comerr-dev curl e2fsprogs krb5-multidev libbrotli-dev libcom-err2 libcurl3t64-gnutls libcurl4-openssl-dev libcurl4t64 libext2fs2t64 libgmp-dev libgmpxx4ldbl libgnutls-openssl27t64 libgnutls28-dev
  libgssrpc4t64 libidn2-0 libidn2-dev libjson-c-dev libkadm5clnt-mit12 libkadm5srv-mit12 libkdb5-10t64 libkrb5-dev libldap-dev libnghttp2-14 libnghttp2-dev libnghttp3-dev libp11-kit-dev libpsl-dev librtmp-dev
  librtmp1 libss2 libssh2-1-dev libssl-dev libtasn1-6-dev libtasn1-doc libtss2-policy0t64 libtss2-tcti-i2c-ftdi0 libtss2-tcti-i2c-helper0 libtss2-tcti-pcap0t64 libtss2-tcti-spi-ftdi0 libtss2-tcti-spi-ltt2go0
  libtss2-tcti-spidev0 libzstd-dev logsave nettle-dev
```

But no, this still doesn't work. Installing the libraries regenerates the `initramfs`, which then shows us our good friend, the error

```text
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
```

### tpm2-tools

So I turn to Google (looking for the exact error message), and I spot [this Reddit post](https://www.reddit.com/r/Fedora/comments/szlvwd/psa_if_you_have_a_luks_encrypted_system_and_a/). I somewhat ignored it before because it was for Fedora, not Debian, but desperate times call for desperate measures. I follow its advice

> After that, make dracut aware of the tpm2 by creating the `/etc/dracut.conf.d/tss2.conf` file, with its contents being the following:
>
> `install_optional_items+=" /usr/lib64/libtss2* /usr/lib64/libfido2.so.* "`
>
> After that, recreate the initramfs by running `sudo dracut -f` and reboot your system. If everything went correctly, your system should automatically boot without the need to input your LUKS password.

I checked `/etc/dracut.conf` and `/etc/dracut.conf.d/` in case the directive was already there, and it wasn't.

```bash
$ cat /etc/dracut.conf
# PUT YOUR CONFIG IN separate files
# in /etc/dracut.conf.d named "<name>.conf"
# SEE man dracut.conf(5) for options
$ ls -l /etc/dracut.conf.d/
total 0
$ echo 'install_optional_items+=" /usr/lib64/libtss2* /usr/lib64/libfido2.so.* "' > /etc/dracut.conf.d/tss2.conf
$ cat /etc/dracut.conf.d/tss2.conf 
install_optional_items+=" /usr/lib64/libtss2* /usr/lib64/libfido2.so.* "
```

and yet

```text
$ sudo dracut -f
dracut[I]: Executing: /usr/bin/dracut -f
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
dracut[E]: Module 'systemd-cryptsetup' depends on module 'tpm2-tss', which can't be installed
[...]
```

A comment below warns that this does not work on Fedora 36 because of some issues with the `tpm2-tools` package, that I don't have.

```text
# apt-cache policy tpm2-tools
tpm2-tools:
  Installed: (none)
  Candidate: 5.7-1
  Version table:
     5.7-1 500
        500 http://deb.debian.org/debian testing/main amd64 Packages
```

Might as well try, right?

```text
# apt-cache policy tpm2-tools
tpm2-tools:
  Installed: (none)
  Candidate: 5.7-1
  Version table:
     5.7-1 500
        500 http://deb.debian.org/debian testing/main amd64 Packages
# apt-get install tpm2-tools
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  tpm2-tools
0 upgraded, 1 newly installed, 0 to remove and 700 not upgraded.
Need to get 758 kB of archives.
After this operation, 1,808 kB of additional disk space will be used.
Get:1 http://deb.debian.org/debian testing/main amd64 tpm2-tools amd64 5.7-1 [758 kB]
Fetched 758 kB in 3s (298 kB/s)     
Selecting previously unselected package tpm2-tools.
(Reading database ... 258880 files and directories currently installed.)
Preparing to unpack .../tpm2-tools_5.7-1_amd64.deb ...
Unpacking tpm2-tools (5.7-1) ...
Setting up tpm2-tools (5.7-1) ...
Processing triggers for man-db (2.13.1-1) ...
# dracut -f
dracut[I]: Executing: /usr/bin/dracut -f
[...]
dracut[I]: *** Including module: systemd-cryptsetup ***
[...]
dracut[I]: *** Including module: tpm2-tss ***
[...]
dracut[I]: *** Including modules done ***
dracut[I]: *** Installing kernel module dependencies ***
dracut[I]: *** Installing kernel module dependencies done ***
dracut[I]: *** Resolving executable dependencies ***
dracut[I]: *** Resolving executable dependencies done ***
dracut[I]: *** Generating early-microcode cpio image ***
dracut[I]: *** Constructing GenuineIntel.bin ***
dracut[I]: *** Store current command line parameters ***
dracut[I]: *** Stripping files ***
dracut[I]: *** Stripping files done ***
dracut[I]: *** Creating image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' ***
dracut[I]: *** Hardlinking files ***
dracut[I]: *** Hardlinking files done ***
dracut[I]: Using auto-determined compression method 'zstd'
dracut[I]: *** Creating initramfs image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' done ***
dracut[I]: *** Moving image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' to '/boot/initrd.img-6.12.35+deb13-amd64' ***
dracut[I]: *** Moving image file '/boot/initrd.img-6.12.35+deb13-amd64.tmp' to '/boot/initrd.img-6.12.35+deb13-amd64' done ***
```

**_WHAT?!_**

## Step 6: The end is near

Let's check that we don't have a problem with the enrollment

```text
# systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+7+11+14" /dev/nvme0n1p3 
🔐 Please enter current passphrase for disk /dev/nvme0n1p3: [REDACTED]
This PCR set is already enrolled, executing no operation.
```

then

```text
# reboot
```

And without touching my keyboard I am brought back to the login screen.

Mission complete :)

## Step 7: What's next

I still need to figure out

1. if my PCR combination will annoy me next time I have a kernel update (which, coincidentally, is _now_, or at least once I have a decent internet connection)
1. if this warrants a bug against `dracut` in Debian (to have a `Depends` or a `Recommends` on `tpm2-tools`)
1. if this was the result of a combination of all the things I tried, or it was really just about `tpm2-tools`

I also need to lock down GRUB; the BIOS is already password protected, but GRUB is probably a weakness if I ever get the combination of PCRs wrong.

For now I'll enjoy my auto-decrypting laptop, and take it from there.

Until next time!

[^dracut]: Reading up on `dracut` is left as an exercise to the reader.
[^pcrregisters]: Figuring out which PCRs to use is also left as an exercise to the reader, but my suggestion is to read the [UAPI Group Specifications of Linux TPM PCR Registry](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/), the [TPM guide for Arch Linux](https://wiki.archlinux.org/title/Trusted_Platform_Module), [this community post](https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005) from the Framework (laptop company) website, and, if you really hate yourself or for some reason really like PDFs, the [TCG PC Client Specific Platform Firmware Profile Specification](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/) from the Trusted Computing Group.
[^eyeball]: I checked the contents of `/lib/modules` and eyeballed it to `6.12.35+deb13`
[^duh]: Which seems obvious since it's a _documentation_ package.
[^rtfm]: my reading the manpage of Dracut, which I should've done way earlier in the process
