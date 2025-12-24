+++
date = '2025-12-24T10:33:34+01:00'
title = 'Site to Site VPN for Dummies between pfSense and a Raspberry Pi'
+++

Today I find myself trying to solve a question that plagued historians for the past few hundred years: why the _fork_ can't I ping my off-site backup server?!

## Background

I put a NAS and a Raspberry Pi in `$secret_location` to my backups (you're following the [3-2-1 Rule](https://en.wikipedia.org/wiki/Glossary_of_backup_terms) too, right?).

The Pi runs Raspbian, with Wireguard configured - via `systemd` + `wg-quick`, and a monitoring cronjob - to connect to my homelab (running pfSense). Throw in a couple static routes, and I can configure my TrueNAS to run a backup job to a private IP over a secure connection. The end, right?

### The actual config

_[Skip this section](#the-issue) if you don't care about the configuration. TBH it's mostly for myself, when I inevitably have to redo this._

#### pfSense

`VPN > Wireguard`, create a new "Site-to-Site" tunnel with its own interface (for simplicity), and **add a new client**: you'll need its IP address later.

You need the client's public key to add the client; you have two choices:

1. `wg genkey | wg pubkey` and then you regenerate it later when you set up the Pi (and update it in pfSense)
2. `wg genkey | tee -a /tmp/client_wg_private_key | wg pubkey` so you can use it later

After creating the client go to `Interfaces > Assignments` and create a new interface called e.g. `SiteToSiteWG`, with static IPv4 and (if appropriate) static IPv6. You don't need DHCP because native Wireguard doesn't do DHCP.

Then go to `Firewall > Rules` and add the appropriate rule(s) to the interface (pretty much "any/any" if you trust everything behind Wireguard, or restrict as necessary).

**The non-obvious part**: navigate to `System > Routing > Gateways` add a new gateway; the interface is the new `SiteToSiteWG`, pick the address family you want[^gwv4v6], and the client IP address is the Wireguard IP you picked at the start. `Disable Gateway Monitoring` to avoid annoyances.

`System > Routing > Static Routes` and add as many static routes as the subnets you want to access in the other site; use your new Gateway as the gateway (obviously).

#### Raspberry Pi

Install `wireguard` (which also gives you `wg-quick` and its `systemd` unit)

You can either reuse the private key you generated before (if you kept it) or `wg genkey` a new one; don't forget to update the public key in pfSense if you do.

With this config

```ini
# cat /etc/wireguard/client.conf 
[Interface]
PrivateKey = aHVudGVyMiAycmV0bnVoIGh1Mm50ZXIgdGVyMmh1bgo=
Address = <WG_IP_IN_PFSENSE>
MTU = 1360

[Peer]
# pfSense
PublicKey = FPkgaCT5mABSgGK/srlovcrPw85w6nRffK/bl7n0jzw=
Endpoint = pfsense.example.com:51280
AllowedIPs = SUBNETS/YOU, WANT/TO, REACH/BEHIND, PFSENSE
PersistentKeepalive = 25
```

check that `wg-quick up client` works, then `systemctl enable --now wg-quick@client` to make this permanent.

Now pfSense and your Pi can talk to each other, but the Pi won't forward packets between the two networks until you `echo 1 > /proc/sys/net/ipv4/ip_forward`. To make this persistent between reboots create `/etc/sysctl.d/10-ip_forward.conf`

```bash
# cat /etc/sysctl.d/10-ip_forward.conf 
net.ipv4.ip_forward = 1
```

If you don't have the `/etc/sysctl.d` folder, you could use the `/etc/sysctl.conf` file, but you really shouldn't (_this is called [foreshadowing](#the-issue)_)...

Optionally, you can set up a cronjob that, if it can't ping your pfSense, will `systemctl restart wg-quick@client`. You can make it as easy or as complicated as you want; here's mine (your guess which way I went between simple and complicated)

```bash
# cat /etc/cron.d/wireguard-monitor 
# m h  dom mon dow user  command
# Every minute root runs the script
* * * * * root  /root/scripts/wg-monitor.sh 2>&1 > /dev/null
```

```bash
# cat /root/scripts/wg-monitor.sh
#!/bin/bash
SERVICENAME="wg-quick@client.service"
PINGADDR="PFSENSE_IP_HERE"
MAXPINGCHECKS=3

DEBUG=$1 # Usage: ./wg-monitor.sh debug
WG_ENABLED=$(systemctl is-enabled $SERVICENAME)
WG_ACTIVE=$(systemctl is-active $SERVICENAME)

# log checks whether we're debugging and decides whether to log to stdin or syslog.
log () {
        LOG_HEADER="wg tunnel monitor:"
        if [ "${DEBUG}" == "debug" ]; then
                echo "$(date "+%F %T") ${LOG_HEADER} ${1}"
        else
                logger "${LOG_HEADER} ${1}"
        fi
}

# Disabled tunnel, don't restart it.
if [ "${WG_ENABLED}" != "enabled" ]; then
        log "tunnel not enabled (${WG_ENABLED})"
        exit 1
fi

# Inactive tunnel. Don't restart it.
if [ "${WG_ACTIVE}" != "active" ]; then
        log "tunnel not active (${WG_ACTIVE})"
        exit 1
fi

# Do 3 ping checks, 30 seconds apart.
for i in $(seq 1 $MAXPINGCHECKS);
do
        ping -c 1 $PINGADDR 2>&1 > /dev/null
        if [ $? -eq 0 ]; then
                # Everything is fine
                log "ping check ok. Exiting."
                exit 0
        fi;
        # Ping failed, wait 15 seconds to make sure it's not a fluke.
        # Wrapped in an if so it doesn't delay the restart.
        if [ $i -ne $MAXPINGCHECKS ]; then
                log "ping check ${i} failed. Waiting 15 seconds."
                sleep 15  # We only wait 15s because a failed ping takes a bit
        fi
done

log "${MAXPINGCHECKS} ping checks failed: restarting tunnel"
systemctl restart $SERVICENAME
```

You're almost done!

#### UniFi

I'm going to assume you have a UniFi gateway at the other end of the network.

Why? Because I do. What if you don't have a UniFi gateway? You can figure out how to do this on your own, it's not complicated :)

The "ugly" part for me was understanding WHY I had to do this[^oopsie]. In hindsight it makes sense, but at the time I was screaming "WHY DON'T YOU WORK?!" over and over again.

Also: **yes, I know UniFi has an integrated Wireguard client**; if I wanted everything routed via the site-to-site I'd have used that. They may allow partial routing now, I didn't really keep up to date with their software, and this works, so no reason to change until I have to.

Anyway, on version `10.0.162` of the UniFi Network app/console:

1. go to `Settings > Policy Engine > Policy Table`, click `Create New Policy` and add a new `Route`, then give it a name.
2. _Type_ `Static`. The _Device_ is `Gateway`, the _Distance_ is `2` and _Next Hop_ is the **LOCAL** IP of the Raspberry Pi (i.e. NOT the IP we gave it in pfSense)
3. the _Destination > Network_ is the subnet you want to reach behind pfSense (must match one of the subnets in the `AllowedIPs` section of your Wireguard config file)

Click `Add` and repeat for all the subnets you need. This tells the UniFi gateway that every time it sees a packet for one of the destination networks it shouldn't try to send it to the internet via the default gateway, but it should send it to the Raspberry Pi, which will take care of it.

And you're done! You now have a shiny new site-to-site VPN.

## The issue

Getting back to the reason I started writing this in the first place: yesterday I updated Raspbian to `trixie`; last night my backup failed. Yay.

Here's what I know[^lies]:

1. I can't ping the remote NAS from my homelab, but I can ping the Pi
2. the Pi can ping the homelab in its entirety
3. the Pi can ping the NAS

I `tcpdump`-ed some traffic and saw that ICMP packets arrived, but there was no response. Crucially, I also didn't see ICMP packets **leaving** the Pi, which told me that the Pi wasn't forwarding packets, and sure enough:

```bash-session
# cat /proc/sys/net/ipv4/ip_forward
0
```

So after a quick

```bash-session
echo 1 > /proc/sys/net/ipv4/ip_forward
```

everything started working again.

Apparently `trixie` got rid of `/etc/sysctl.conf`?[^sysctlconf] `/etc/sysctl.conf.dpkg-bak` exists, and has my `ip_forward` config in it (from when I initially set this up, many moons ago), but `sysctl.conf` is gone.

I don't know if this is WAI or just a fluke, but since using dedicated files seems to be the new way to go I just created the `/etc/sysctl.d/10-ip_forward.conf` file to ensure persistence

```conf
# cat /etc/sysctl.d/10-ip_forward.conf 
net.ipv4.ip_forward = 1
```

And hopefully everything should keep working from now on.

[^gwv4v6]: To do both IPv4 and v6 you need two separate gateways
[^oopsie]: When I originally set this up this step took me an embarrassingly long time to understand: I could see (with `tcpdump`) packets leaving my homelab, arriving on the Pi, leaving the Pi (being forwarded), and arriving on the destination server; the server replied (to e.g. a `ping`) and its packets would be leaving the destination server, but NEVER reaching the Pi. That's because UniFi was like "oh I don't know this network, better send it to the internet!".
[^lies]: I just want to point out that this took about 1 hour to figure out: I rebooted both Pi and pfSense before even thinking about `tcpdump`, which I had to install. I knew the NAS was alive because the Pi could ping it, and something in the back of my head suggested the `ip_forward` setting could be the issue, but of course I dismissed it without even checking it because "it was working before, that's a system setting, there is NO WAY it changed". Checking it would've saved me between half an hour and 45 minutes of troubleshooting...
[^sysctlconf]: According to `man sysctl.conf` (bold is mine)
    > **FILES**
    >
    > `procps sysctl`, when run with the `--system` option, reads files from directories in the order shown below.
    >
    > [...]
    >
    > Finally, procps sysctl reads `/etc/sysctl.conf`. **This file is not used by `systemd-sysctl`, which means that some kernel parameters re not set depending on the implementation of sysctl that is installed**.

    This may explain why they're trying to get rid of it? Thanks `systemd`!
