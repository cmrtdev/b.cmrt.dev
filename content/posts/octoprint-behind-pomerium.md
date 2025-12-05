+++
date = '2025-12-05T22:36:27+01:00'
draft = false
title = 'Hiding Octoprint behind Pomerium and dealing with WebSockets'
+++

I run [OctoPrint](https://octoprint.org/) behind [Pomerium](https://www.pomerium.com/). The actual configuration is somewhat irrelevant (although I'll still paste it here)

```yaml
  # Octoprint
  - from: https://octoprint.example.com
    to: http://192.168.169.170
    allow_websockets: true
    policy:
      - allow:
          or:
            - email:
                is: me@example.com
    preserve_host_header: true
```

but what you should know is that OctoPrint really likes its WebSocket and will refuse to connect without it.

In the network console you'll see attempts to connect to `wss://octoprint.example.com/sockjs/123/random_string/websocket` all returning 403 `Access Denied` instead of 101 `Switching Protocols`.

Your `~/.octoprint/logs/tornado.log` will tell you that

```text
2025-12-05 12:06:17 - tornado.access - WARNING - 403 GET /sockjs/057/lrphguyk/websocket (192.168.11.12) 3.42ms
2025-12-05 12:06:23 - tornado.access - WARNING - 403 GET /sockjs/561/fn4lzccs/websocket (192.168.11.12) 6.27ms
2025-12-05 12:08:00 - tornado.access - WARNING - 403 GET /sockjs/963/2jkjryyv/websocket (192.168.11.12) 23.46ms
2025-12-05 12:08:01 - tornado.access - WARNING - 403 GET /sockjs/497/rftixzbz/websocket (192.168.11.12) 18.54ms
2025-12-05 12:08:40 - tornado.access - WARNING - 403 GET /sockjs/607/yh0u5hw1/websocket (192.168.11.12) 7.46ms
2025-12-05 12:08:42 - tornado.access - WARNING - 403 GET /sockjs/002/lou5nx24/websocket (192.168.11.12) 3.67ms
```

and even if you enable Tornado debug logging[^debug] all you'll find out (in `octoprint.log`, not `tornado.log`) is that

```text
2025-12-05 12:08:42,027 - tornado.general - DEBUG - Cross origin websockets not allowed
```

However, a Google search of that error will bring you to [this OctoPrint community post](https://community.octoprint.org/t/web-socket-issue-exposing-octoprint-using-cloudflared-formerly-argo/38052) where `kelzebub` tells you

> I found that in the octoprint settings, under API, checking this box solves this issue.
>
> > Allow Cross Origin Resource Sharing (CORS)

while `Randy_Lai` gives you the instructions to do the same thing, but in the config file

> Setting the following in config.yaml
>
> ```yaml
> api:
>   allowCrossOrigin: true
> ```
>
> indeed fixed the connection issue.

Now, here's where I remind you that **you should not do this unless you know what you're doing**, and especially if your OctoPrint is **exposed to the internet**.

Blindly enabling CORS[^corswiki][^corsmdn] with no protection (e.g. the [`Access-Control-Allow-Origin`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin) header) is a bad idea and I am not responsible for anything that may happen to you, your OctoPrint, printer(s), possessions, etc. :)

It would be awesome if OctoPrint supported specifying allowed CORS origins, but for now this will do.

### But what happened?

I don't actually know what happened: a while ago I started getting warnings from OctoPrint that I was running an ancient version of Python and, rather than dealing with an `apt-get dist-upgrade` from `bullseye` to `bookworm` and then `trixie`, I just decided to reinstall OctoPi[^octopirecommended].

After reinstalling it and restoring from backup, I couldn't access it anymore, and I got an error page saying _the Socket connection failed_ (or something like that).

Fortunately I still had direct IP access, which was working fine. I actually narrowed down the issue to Tornado pretty fast, but then went down the rabbit hole of trying to figure out if this was because my `Host` header did not include the port (see [this GitHub issue](https://github.com/OctoPrint/OctoPrint/issues/3193) to understand what I mean), and I managed to force Pomerium to return the port as part of the Host header[^hostrewrite], but that didn't work.

I thought I was doing something wrong, because Pomerium seemed to be sending the correct `host:port` combination, OctoPrint's `reverse_proxy_test` page was all green (both in host and port) and Tornado was still 403-ing. I almost went and hot-patched OctoPrint's code to make it log the Host header value it was seeing and the one it was expecting. In the end I didn't, because enabling CORS fixed my issue and my OctoPrint is fairly well protected (and not on the internet).

Going forward I'm not sure what I'll do: I wonder if I should dig a little bit more or not, open a bug or not. Maybe over the holidays, if I have time; in the meantime, I hope this helps someone else.

[^debug]: In OctoPrint's UI: Settings > Logging > Logging Levels, add the `tornado` logger at DEBUG level
[^corswiki]: <https://en.wikipedia.org/wiki/Cross-origin_resource_sharing>
[^corsmdn]: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS>
[^octopirecommended]: Which, at the time of writing, is [the officially recommended way to upgrade Python for OctoPrint](https://community.octoprint.org/t/octoprint-tells-me-my-python-environment-is-outdated-and-will-soon-no-longer-be-supported-how-do-i-upgrade/61076)
[^hostrewrite]: By using the [`host_rewrite` option](https://www.pomerium.com/docs/reference/routes/headers) to set `octoprint.example.com:443` as the _Host_ value, and then checking using a locally deployed version of [httpbin](https://httpbin.org/), protected by the same Pomerium, with identical settings
