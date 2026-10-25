# Give your Pi-hole machine a static IP

Pi-hole needs an address that never changes. Pick one of these, then use that IP everywhere in the video.
Replace `10.37.0.41/24` with your address, `10.37.0.1` with your router (gateway), and `eth0` / `ens18` with your
interface name (find both with `ip -br a` and `ip r`). If it shows something like `eth0@if68` (containers do this),
the name is just the part before the `@`.

**Easiest, works for every distro:** create a DHCP reservation for this machine in your router's DHCP settings, so
the router always hands it the same address. Nothing to change on the machine.

## Debian (ifupdown) — what the video shows
`/etc/network/interfaces`:
```
auto eth0
iface eth0 inet static
    address 10.37.0.41/24
    gateway 10.37.0.1
```
Then: `sudo systemctl restart networking`

DNS isn't set in this file. Debian keeps it in `/etc/resolv.conf`, which your DHCP client already filled in, so it keeps
working after the switch. Check it:
```
cat /etc/resolv.conf
```
If there's no `nameserver` line, add one (otherwise the Pi-hole machine itself can't look up names for updates):
```
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```
Exception: if the `resolvconf` package is installed (`dpkg -l resolvconf`), don't edit resolv.conf by hand, it gets
rewritten. Add `dns-nameservers 1.1.1.1` under the `gateway` line instead, then restart networking. If the machine uses
NetworkManager (desktop installs, Raspberry Pi OS Bookworm), use the nmcli section below instead.

## Ubuntu Server (netplan)
`/etc/netplan/50-cloud-init.yaml` (name may differ, check `ls /etc/netplan`):
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses: [10.37.0.41/24]
      routes:
        - to: default
          via: 10.37.0.1
      nameservers:
        addresses: [1.1.1.1]
```
Then: `sudo netplan apply`

## Raspberry Pi OS (Bookworm), Fedora, RHEL / Rocky / Alma, Ubuntu Desktop (NetworkManager)
```
nmcli con show                       # find your connection name, e.g. "Wired connection 1"
sudo nmcli con mod "Wired connection 1" ipv4.method manual \
  ipv4.addresses 10.37.0.41/24 ipv4.gateway 10.37.0.1 ipv4.dns 1.1.1.1
sudo nmcli con up "Wired connection 1"
```

## Arch (systemd-networkd)
`/etc/systemd/network/20-wired.network`:
```
[Match]
Name=eth0

[Network]
Address=10.37.0.41/24
Gateway=10.37.0.1
DNS=1.1.1.1
```
Then: `sudo systemctl restart systemd-networkd`

## Raspberry Pi OS (Bullseye and older, dhcpcd)
Add to `/etc/dhcpcd.conf`:
```
interface eth0
static ip_address=10.37.0.41/24
static routers=10.37.0.1
static domain_name_servers=1.1.1.1
```
Then: `sudo reboot`
