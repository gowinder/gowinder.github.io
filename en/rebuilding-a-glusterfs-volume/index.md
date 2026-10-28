# Rebuilding a GlusterFS Volume


# Rebuilding a GlusterFS Volume

## Background

When I first set up the `glusterfs` cluster I was in a hurry and used the network addresses of the `pve` hosts directly. After a few days I noticed that when `volume` replication ran it was saturating the `pve` hosts' own network channel. The `dell r720` happened to have four NICs and the `e3` host had two, so I took one from each and dedicated it as the `gluster` bind address.

But then, without thinking, I simply changed the `ip` for `gluster1` and `gluster2` in the `hosts` files on both sides — and it didn't work: it kept using the old addresses, and `gluster1` kept reporting `gluster` as offline. So I decided to delete the `volume` and reconfigure it.

## Process

I first asked the new Bing how to delete a `volume`; it gave me the steps, but I found one of them was in the wrong order. I tried them myself, in the order below:

- On both sides, stop the `volume`: `sudo gluster vol stop VMS`
- On both sides: `sudo umount gluster1:/tank/gfs/s`
- On both sides delete the `brick`: `sudo gluster vol remove-brick VMS replica 1 gluster2:/tank/gfs/s force` — note: on `gluster2` remove `gluster1`, and on `gluster1` remove `gluster2`
- `sudo gluster vol delete VMS start`
- This step errors out saying a `peer` still exists, because on `gluster1` `gluster2` always shows as `disconnect`, so the `peer` has to be removed
- Check the `peer`: `sudo gluster peer status`, which shows `disconnected`
- On both sides remove the `peer`: `sudo gluster peer detach gluster2`
- `sudo gluster peer status` again shows no `peer` remains
- Delete the `volume`: `sudo gluster volume delete VMS`, which reports success

### If it errors when re-joining

Do this on both nodes:

- `sudo systemctl stop glusterd`
- `peer probe: failed: g1 is either already part of another cluster or having volumes configured` — you need to, on the `g1` host, delete everything under `/var/lib/glusterd` except the `glusterd.info` file, then run `sudo glusterd peer probe g1`
- Delete `/tank/gfs/s/.glusterfs`

But it still errored saying `/tanks/gfs/s` was already part of a `volume`, so I had no choice but to create a new directory.

**Important**: with the new directory you must `mount` it first, then copy into the mounted directory — e.g. `sudo mount g1:vms`, and after it is mounted at `/vms`, copy into `/vms`. Otherwise it will not sync. See: [stackoverflow](https://stackoverflow.com/a/29301051)

## Summary

A `gluster` node's address cannot be changed casually — you have to set it with commands.

### The correct approach (untested)

- Stop the `volume`: `sudo gluster volume stop VMS`
- `sudo umount /vms`
- `gluster volume set VMS config.transport tcp`

- ```shell
  gluster volume set VMS config.transport.tcp.bind-address newaddress
  gluster volume set VMS config.transport.socket.bind-address newaddress
  gluster volume set VMS config.transport.rdma.bind-address newaddress
  ```

- `sudo gluster volume start VMS`
- `sudo mount /vms`

