# Fedora CSB Laptop Setup

Last verified: 2026-09-08

## System

- Hostname: `dosmith-thinkpadp1gen8.rmtusvt.csb`
- User: `dosmith`
- Operating system: Fedora Linux CSB 44
- LAN address: `192.168.50.196/24`
- LAN gateway and DNS: `192.168.50.1`
- Wi-Fi connection: `JoinerBrook 1`
- Wi-Fi MAC used by the router reservation: `3A:E5:93:A4:56:BD`
- Hardware: 16 logical CPUs, 62 GiB RAM, and approximately 943 GiB free in `/home`
- Intel VT-x and `/dev/kvm` are available

The router has a DHCP reservation mapping the Wi-Fi MAC above to
`192.168.50.196`. Keep the laptop connection on DHCP; do not also configure a
static address in NetworkManager.

## CSB compliance

The laptop was reinstalled with the Fedora CSB 44 image after the shipped
Fedora 43 installer failed to find the Red Hat user. The IPA password was reset
at `https://identity.corp.redhat.com/resetipa` after an initial Kerberos
pre-authentication failure.

The Fleet application reports compliant for all policies. Work was tracked in
`INFERENG-9781`.

## Remote access

SSH is allowed in the CSB-supported firewalld `work` zone for the private home
Wi-Fi connection:

```bash
ssh dosmith@192.168.50.196
```

GNOME Remote Login is enabled as a system service on RDP port `3389`. It starts
at the GDM login screen and does not require an existing interactive GNOME
session. GNOME Desktop Sharing remains configured separately for an existing
user session and negotiates another port when Remote Login owns `3389`.

On `yoda`, the machine-local helper script `~/connect-csb-laptop.sh` launches
Remmina. It prefers the home-LAN endpoint `192.168.50.196:3389` and falls back
to the tailnet Serve endpoint on port `3390`. The fallback only works while
Tailscale is enabled on both systems.

```bash
~/connect-csb-laptop.sh
```

Remmina first requests the configured GNOME Remote Login RDP credentials. The
GDM screen then handles the actual `dosmith` system login. The RDP service is
allowed only through the CSB-supported firewalld `work` zone used by the home
Wi-Fi connection.

## NetBird and Tailscale

NetBird and Tailscale cannot be healthy simultaneously on this laptop. Both
claim `100.100.100.100` for DNS, and Tailscale's anti-spoofing rules interfere
with NetBird traffic in the shared CGNAT address range.

The normal work state is NetBird connected and Tailscale down:

```bash
sudo tailscale down
sudo netbird up
```

For tailnet access, disconnect NetBird before enabling Tailscale:

```bash
sudo netbird down
sudo tailscale up
```

Switch back to NetBird for Red Hat access:

```bash
sudo tailscale down
sudo netbird up
```

NetBird is split-tunneled: ordinary internet traffic uses the LAN gateway,
while Red Hat/private routes and corporate DNS use NetBird. Tailscale address:
`100.122.147.117`; NetBird address: `100.91.166.185`.

## Power modes

Power-mode helpers live in `~/laptop-mode`:

```bash
~/laptop-mode/laptop-mode status
~/laptop-mode/laptop-mode always-on
~/laptop-mode/laptop-mode mobile
```

`always-on` enables `laptop-always-on.service`, which inhibits lid-triggered
suspend. Closing the lid may still end GNOME Desktop Sharing because it is tied
to the interactive GNOME session.

## Codex

Codex CLI is installed and configured for `dosmith`. Authentication, config,
rules, and memories were transferred from `yoda` and smoke-tested. Do not put
credentials or tokens in this document.

## Vagrant and libvirt

The laptop supports KVM. Vagrant, `vagrant-libvirt`, `virt-install`, and the
remaining libvirt dependencies are installed from Fedora packages. `libvirtd`
and the libvirt NAT network are enabled, and `dosmith` belongs to the `libvirt`
group.

The `agent-fedora` project lives at `~/vagrant/agent-fedora`. It uses the
official Fedora 44 Cloud Vagrant image for libvirt, verified with the SHA-256
published in Fedora's signed checksum file. The VM has 4 vCPUs, 16 GiB RAM, a
100 GiB thin-provisioned disk, and a libvirt NAT interface. Common development
tools (`git`, `jq`, `ripgrep`, `tmux`, and Vim) are installed while provisioning.

The VM is also enrolled in Doug's tailnet as `agent-fedora`, with Tailscale
address `100.84.226.6`. The normal direct-access path from another tailnet
machine is:

```bash
ssh dev@agent-fedora
```

The `dev` user's authorized keys include access from Doug's machines. Its SSH
permissions must remain `0700` on `~dev/.ssh` and `0600` on
`~dev/.ssh/authorized_keys`; a directory mode of `0644` prevents traversal and
causes sshd to fall back to password authentication.

Vagrant remains the recovery and local-management access path from the laptop:

```bash
cd ~/vagrant/agent-fedora
vagrant status
vagrant ssh
vagrant halt
vagrant up
vagrant destroy -f   # permanently deletes this VM
```

System-libvirt images live in the autostarted `agent-vms` pool at
`/var/lib/libvirt/images/agent-vms`.

### LAN bridging

A direct macvtap NIC over `wlp0s20f3` was tested. System libvirt could create
it and the guest saw the device, but the home access point did not pass DHCP
traffic for the VM's additional MAC address. The final configuration therefore
uses reliable NAT networking.

Because `ssh dev@agent-fedora` provides direct tailnet access while Vagrant
retains a local management path, bridged networking is not currently required.

For true guest addresses on `192.168.50.0/24`, attach a wired or USB-C Ethernet
adapter, create a NetworkManager bridge such as `br0`, and configure Vagrant's
`public_network` against that bridge. This matches the bridged-network design
already used by the DRA lab Ansible role.
