# Resolving Gluster Brick Offline Issues


# Resolving Gluster Brick Offline Issues

## Offline Issue

I had just finished installing `gluster` and stored the `rootfs` of a `linux container` on it, so that the `linux container` could achieve high availability within a two-node `pve` cluster.

When I had some spare time, I checked with the command: `sudo gluster vol status VMS`, and found that one `brick` was in `offline` state.

I looked it up online and installed `gstatus`:

```shell
curl -fsSL https://github.com/gluster/gstatus/releases/latest/download/install.sh | sudo bash -x
gstatus --version
```

After the installation was complete, I ran: `sudo gstatus -a -b` and found that: `gluster1` was in `offline` state. I searched around and found a solution: [force start](https://github.com/gluster/glusterfs/issues/2742#issuecomment-908361146), and followed it:

```shell
sudo gluster vol start VMS force
```

Shortly after running it, everything was back to normal.

## Heal Issue

While inspecting `gluster`, I found that a `heal` was needed,

- Check: `sudo gluster heal VMS info` to see whether anything needs to be `heal`ed
- `heal`: `sudo gluster heal VMS`,
- View summary: `sudo gluster vol heal VMS summary`
- View statistics: `sudo gluster vol heal VMS statistics heal-count`

