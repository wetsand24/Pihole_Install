# Pi-hole: block ads on every device in your house

Every command and setting from the video, in order. Replace `10.37.0.41` with your Pi-hole machine's IP and
`10.37.0.1` with your router.

▶️ Video: https://youtu.be/k9GPXPp1KU0

## What you need
- Any Debian or Ubuntu machine: a Raspberry Pi, an old laptop, a VM or an LXC container (1 core, 512 MB RAM is plenty)
- A static IP for it (step 1)

## 1. Give the machine a static IP
Check its current address and your router (gateway):
```bash
ip -br a
ip r
```
Your interface name is the first column of `ip -br a` (e.g. `eth0`, `ens18`, `enp0s3`). In the video it shows
`eth0@if68` because the Pi-hole runs in a Proxmox container: `@if68` just points to the other end of its virtual
cable on the host. The name to use in the config is still `eth0`. Use your own name wherever you see `eth0` below.

**Debian** (what the video shows). Edit the network file:
```bash
nano /etc/network/interfaces
```
Change `iface eth0 inet dhcp` to static and add your address and gateway:
```
auto eth0
iface eth0 inet static
    address 10.37.0.41/24
    gateway 10.37.0.1
```
Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`, then:
```bash
systemctl restart networking
ip -br a
```
**DNS for the machine itself:** Debian keeps it in `/etc/resolv.conf`, which your DHCP client already filled in. Check it:
```bash
cat /etc/resolv.conf
```
No `nameserver` line? Add one: `echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf`.
If the `resolvconf` package is installed, add `dns-nameservers 1.1.1.1` under the `gateway` line instead.

**Ubuntu, Fedora, RHEL, Arch, Raspberry Pi OS:** see [STATIC-IP-GUIDE.md](STATIC-IP-GUIDE.md).
Easiest option for any distro: a DHCP reservation for this machine in your router.

## 2. Install Pi-hole
```bash
apt update && apt upgrade -y
apt install -y curl
curl -sSL https://install.pi-hole.net | bash
```
In the installer: pick an upstream DNS provider (any is fine), keep the default blocklist, and say **Yes** to query logging.

Set the web password:
```bash
pihole setpassword
```
Dashboard: `http://10.37.0.41/admin`

## 3. Point your PC at Pi-hole (Windows)
Test first. Before the switch, this returns a real address:
```
nslookup doubleclick.net
```
1. `Win + R` → `ncpa.cpl` → Enter
2. Double-click your adapter → **Properties**
3. **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**
4. **Use the following DNS server addresses** → Preferred: `10.37.0.41` → OK → Close

Then:
```
ipconfig /flushdns
nslookup doubleclick.net
```
The answer now comes from `pi.hole` and it's `0.0.0.0`, which means blocked.

> If your PC also has IPv6 DNS from the router, it can bypass Pi-hole. Either set the IPv6 DNS to Pi-hole's IPv6
> address too, or turn on Pi-hole's DHCP/router DNS for the whole house (step 6).

## 4. Better blocklists
Pi-hole web UI → **Lists** → add:
```
https://big.oisd.nl
```
Then **Tools → Update Gravity**. Restart your browser so it forgets cached answers.

Ad block test used in the video: https://adblock.turtlecute.org/

## 5. Parent mode: block a site for the kids only
1. **Groups** → add a group called `Kids`
2. **Clients** → add the kids' device by IP, put it in `Kids` and take it out of `Default`
3. **Domains** → `tiktok.com`, tick **Add domain as wildcard**, **Add to denied domains**
4. In the domain list, set that rule's group to `Kids` only → **Apply**

Check from the kids' device:
```bash
nslookup tiktok.com    # 0.0.0.0 = blocked
nslookup google.com    # still works
```
Give the kids' device a fixed IP (DHCP reservation in the router) so the rule keeps matching it.

## 6. Every device in the house
In your router's DHCP settings, set the **DNS server** to your Pi-hole's IP. Every device picks it up on its next
lease (or reconnect). Keep a second option in mind: if the Pi-hole machine goes down, DNS goes down with it.

## What Pi-hole can't block
YouTube ads. They come from the same servers as the videos. Use a browser ad blocker for YouTube.

## Quick tips
- **Name instead of IP:** Settings → Local DNS Records → `pihole.home` → `10.37.0.41` → Add. Then open `http://pihole.home/admin`.
- **Site acting up?** Sidebar → **Disable Blocking** → *For 5 minutes*. It turns itself back on.

## Links
- Pi-hole: https://pi-hole.net · docs: https://docs.pi-hole.net
- OISD blocklists: https://oisd.nl
