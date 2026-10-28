# A Problem Caused by a YouTube App Using QUIC

---

# A Problem Caused by a YouTube App Using QUIC

## Problem Description

At home, an `r2s` serves as the bypass router using `sing-box` in `tun` mode. I had always used a `cn2 gia` line, and every night I'd watch videos on `youtube` on my `apple tv` before going to sleep, with no problems. This month the server's traffic ran out, so I switched to `hysteria2` on an ordinary line. It was fine all day, but come evening `youtube` was completely unwatchable. I figured `udp` packet loss was probably severe in the evening, so I didn't bother with it.

One day I couldn't take it anymore. In the evening I opened a browser on my laptop and tested the speed at `speed.cloudflare.com`, and it was indeed normal—downstream over `40M`. I figured there'd be no problem watching YouTube with that, so I opened `youtube` in the laptop browser and found the speed normal, 20K+ `Kbps`, and even `4k` was fine. I hurried back to test on the `apple tv`, and it was still laggy, with the speed only 200–300 `Kbps`. I was completely baffled.

## Troubleshooting Process

So I tested with various devices and found that, apart from the `youtube app` on `apple tv` and `chromecast tv`, everything else—phone `app`s and browsers—was completely normal.
This was clearly a problem. I also thought it had only appeared recently, so I googled to see whether anyone else had the same issue. I searched separately by the version numbers of the `youtube app` on `apple tv` and `chromecast tv`, but found nothing relevant.
  
So I looked at the `sing-box` `dashboard`. When playing at normal speed, the connections looked normal; when playing with `youtube tv` on the TV, the connection speed just wouldn't come up and stayed below `100KBps` the whole time.
It was extremely frustrating. I tried switching the protocol to `vmess`, and it was actually faster than the current setup, though the speed in the browser and `app`s still wasn't great.

Just as I was about to give up again, I noticed that on the `dashboard` the number of closed connections for `apple tv` kept growing, while the corresponding `pc` `ip` was normal. This was clearly a problem—how could `apple tv` initiate so many connections? Looking more carefully, I found that the connection from `apple tv` to `YouTube` had the protocol `tun(udp)`. It's clearly video going over port `443`, so how come it's `udp`? I suddenly thought of `quic`. Could it be going over `quic`? And `quic` is based on `udp`, packet by packet, and when the `tun` side receives packets it forwards them to `hysteria`'s port hopping—each (or several) packets change ports and establish new connections, which explains why there were so many "closed" connections.

So I googled and found an [issue](https://github.com/SagerNet/sing-box/issues/664), and tried using this `rule`:

```json
{
"protocol": [
"quic"
],
"outbound": "block"
},
```

Updated the config, and the result... it didn't work. Ugh, it was almost 1 a.m. by then, so I decided to call it a day and try again tomorrow. Because I'd had coffee in the afternoon and kept thinking about this, I didn't sleep well all night. I got up early on the weekend to try again.

## Solution

After thinking it over all night, I felt my reasoning was right: it must be using `quic`, and if I disabled it, the `youtube app` would switch to `tcp`.
I looked at the `dashboard` and found that the `rule` corresponding to the `tun(udp)` entry wasn't the `block` I had set—meaning it hadn't taken effect.

So I kept reading the `sing-box` docs and switched to a different `rule` config:

```json
{
  "type": "logical",
  "mode": "and",
  "rules": [
    {
      "network": ["udp"]
    },
    {
      "port": [443]
    }
  ],
  "outbound": "block"
}
```

This one is very clear: `udp` on port `443` gets `block`ed. It might cause some false positives, but it shouldn't let YouTube's `quic` slip through anymore.

Updated the config, applied the settings, and tested:
![alt text](youtube-speed.png)
Finally it was normal. If I hadn't tested it with the browser, I'd have stayed in the dark all along. Now, at last, I can watch videos and sleep with peace of mind...

