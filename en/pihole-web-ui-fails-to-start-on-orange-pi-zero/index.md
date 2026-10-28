# Pi-hole Web UI Fails to Start on an Orange Pi Zero


# Pi-hole Web UI Fails to Start on an Orange Pi Zero

## Background

I recently installed `Pihole` on an idle `Orange Pi Zero` to use as the home primary `DNS` server. The install went smoothly: `Armbian` -> `Pihole`. But after it finished, the `WebUI` would not start — `Pihole` uses `Lighttp`.

## Troubleshooting

I checked with: `sudo lighttpd -t -f /etc/lighttpd/lighttpd.conf` — the result was `Syntax OK`, no problem.

Checking the `lighttpd` service: `sudo systemctl status lighttpd`, I found it exited immediately on start with `exited=255`.

Looking further into the logs: `sudo tail -f /var/log/lighttpd/error-pihole.log` showed the error:

```shell
2023-02-26 21:53:03: server.c.1513) server started (lighttpd/1.4.59)
2023-02-26 21:53:03: mod_accesslog.c.612) opening log '/var/log/lighttpd/access.log' failed: Permission denied
```

It said the `/var/log/lighttpd/` directory had no permission, so I did the following:

```shell
sudo chown pihole:pihole /var/log/lighttpd -R
sudo chmod 755 /var/log/lighttpd -R
```

Start it again:

```shell
sudo systemctl start lighttpd
```

And it was fine.

