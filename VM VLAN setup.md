

# VLAN Lab: Build Log (OPNsense + client1)

This file contain steps used to setup OPNsense and Alphine VM on VMware

---
## Part 1: OPNsense

### 1.1 Download

Download Link: https://opnsense.org/download/
	- For VMWare Workstation Pro: `amd65` + `dvd`



### 1.2 VM Setup

After adding ISO image and adding the VM, make sure these settings are configured properly in `Settings`

| Setting   | Value                                                                                                         | Why                |
| --------- | ------------------------------------------------------------------------------------------------------------- | ------------------ |
| Firmware  | **UEFI** (settings→Options→Advanced)                                                                          | See issue #1       |
| Memory    | **3000 MB+** (settings)                                                                                       | See issue #3       |
| Disk      | 20 GB+ (pre-config setup)                                                                                     | ZFS wants headroom |
| Adapter 1 | NAT (VMnet8)                                                                                                  | → `em0`, WAN       |
| Adapter 2 | Custom: **VMnet2** (settings → Add... → Network Adaptor → Finish)<br>Set the new adaptor to Custom → `VMnet2` | → `em1`, LAN trunk |
**There must be TWO network adapters before booting up OPNsense.**

### 1.3 Boot Setup

Simply boot and don't press any keys. Once boot has finished, it should ask for login.

Login: `installer`
Password: `opnsense`

A GUI should show and follow these steps:
- Install (ZFS) → stripe → select disk → confirm
- Set root password
- Reboot

Confirm it actually installed — the console menu looks identical on the live ISO and a real install:
```
mount | grep ' / '
```
- Should show `zroot/ROOT/default`. If it says `/dev/iso9660/...` still on the live image.

---

### 1.4 Assign interfaces

Console option **1) Assign interfaces**:

- LAGG: `n`
- VLAN: `n` (done on the web)
- WAN: `em0`
- LAN: `em1`
- Enter to finish, `y` to confirm

Expected result on the banner (example):
```
LAN (em1) -> v4: 192.168.1.1/24
WAN (em0) -> v4/DHCP4: 192.168.48.135/24
```

The WAN address comes from VMware's NAT DHCP on VMnet8. Correct.

Then chose option **2) Set interface IP address**
- Pick LAN
- DHCP: `n`
- Enter `10.10.99.1`
- Subnet bits: `24`
- Upstream gateway: `no`
- IPv6: `no`
- **enable DHCP** with range `10.10.99.100` to `10.10.99.199`. 
	- DHCP will only hand out IP address `.100`-`.199` - apply for all client DHCP
	- For other ranges, must be assigned statically or via reservation (bind by device MAC address)
- Decline the HTTP revert prompt.

Login: `root`
Password: `opnsense`

---

## Part 2: client1 (Alpine)

### 2.1 Create the VM

Download Link: https://alpinelinux.org/downloads/
- Download Virtual x86_64

After adding ISO image and adding the VM, make sure these settings are configured properly in `Settings`

| Setting  | Value                             |
| -------- | --------------------------------- |
| Guest OS | Other Linux **6.x** kernel 64-bit |
| Memory   | 512 MB                            |
| Disk     | 8 GB, thin provisioned            |
| Adapter  | Custom: **VMnet2**                |

---

### 2.2 Install to disk

Running from the live ISO means everything is lost on reboot — including VLAN config.

```
setup-alpine
```

| Prompt               | Answer                                                                   |
| -------------------- | ------------------------------------------------------------------------ |
| Keymap               | `us` / `us`                                                              |
| Hostname             | `client1`                                                                |
| Interface            | `eth0`                                                                   |
| IP                   | `dhcp`                                                                   |
| Manual config        | `n`                                                                      |
| Root password        | *(set one)*                                                              |
| Proxy                | `none`                                                                   |
| Mirror               | `f` (fastest)                                                            |
| SSH                  | `openssh`                                                                |
| Disk                 | `sda`                                                                    |
| Mode                 | **`sys`** - it writes a real install. `data` or `none` keeps you in RAM. |
| Erase the above disk | `y`                                                                      |
|                      |                                                                          |
**Then:**
1. `reboot`
2. Disconnect the ISO 
3. run `apk add vlan tcpdump`

**Shutting off and starting client1 vm should preserve network config**


## Part 3: Clone Client1

### 3.1 Clone

1. Right click on Client1 VM → Manage → Clone (make sure to select Full Clone)
2. Simply rename the VM to `Client 1` and repeat the same for `Client2`

After cloning, boot each one and change its hostname:
```
echo client2 > /etc/hostname
hostname -F /etc/hostname
```


### Summary

### What this build proves

Every layer beneath VLAN tagging is now verified independently. When tagging fails later, the fault is in the tagging — not in six unverified things below it.

| Layer | Verified by | Evidence |
| --- | --- | --- |
| Virtual NIC type | `.vmx` reads `e1000` | Tags will pass through unstripped |
| VMnet2 segment | Client reaches OPNsense | Trunk carries frames end to end |
| OPNsense routing | WAN holds a NAT-range address | Uplink functional |
| DHCP service | Client pulled `10.10.99.x` | Firewall is authoritative on the segment |
| Management access | Web UI answers on `10.10.99.1` | Config surface reachable |
| Persistence | Root on `/dev/sda3`, not `tmpfs` | Survives reboot |

### Current state

| Item | Value |
| --- | --- |
| OPNsense version | 26.7 amd64 |
| WAN | `em0`, DHCP from VMnet8 NAT |
| LAN / trunk | `em1`, `10.10.99.1/24` |
| DHCP pool | `10.10.99.100` – `10.10.99.199` |
| Clients | client1/2/3, Alpine 3.24, VMnet2 |
| VLANs configured | none yet |


### One firewall per segment

VMnet2 can host exactly one router and one DHCP server. For another OPNsense, use another adaptor (VMnet3).

### Next: VLAN configuration

All web UI work, at `https://10.10.99.1` from any client.

1. **Interfaces → Devices → VLAN** — three devices, parent `em1`, tags 10 / 20 / 30
2. **Interfaces → Assignments** — add each, name TRUSTED / IOT / GUEST
3. **Interfaces → [each]** — enable, static IPv4, `10.10.10.1/24`, `10.10.20.1/24`, `10.10.30.1/24`
4. **Services → DHCPv4 → [each]** — enable, `.100`–`.199` per VLAN
5. **Firewall → Rules → [each]** — block RFC1918 and This Firewall *above* the permissive pass rule

---

## 4.0: VM VLAN Interface Setup

Once all four VMs are installed and configured, the next step is setting up the 3 clients interfaces on OPNsense.

All config will be done in the web UI at `https://10.10.99.1`, reached from any client on VMnet2.
- Setup a Window 11 machine or a ISO with graphical interface to visit `https://10.10.99.1` directly on the web

OPNsense splits VLAN setup into three distinct stages. Creating a VLAN device is not the same as assigning it, and assigning it is not the same as configuring it. Skipping a stage leaves the lab silently dead.

---

### 4.1 Create the VLAN devices

**Interfaces → Devices → VLAN** → click **+** three times.

|Parent|VLAN tag|Description|
|---|---|---|
|`em1`|10|Trusted|
|`em1`|20|IoT|
|`em1`|30|Guest|

Leave VLAN priority at **Best Effort (0, default)**. Save each entry, then click **Apply** at the bottom of the list.

Result: three devices named `vlan01`, `vlan02`, `vlan03`.

> **Device number is not the tag.** `vlan01` carries tag 10, `vlan02` carries 20, `vlan03` carries 30. The device number is just a sequential counter OPNsense assigns. Always read the Description and Tag columns, not the device name.

All three share `em1` as parent. That is what makes `em1` a trunk — a trunk is not a setting you enable, it is what an interface becomes when it carries more than one VLAN's tagged traffic.

![[Pasted image 20260920175237.png]]

---

### 4.2 Assign the devices as interfaces

**Interfaces → Assignments**

The three new devices appear in the dropdown at the bottom of the page. Add each one and set its Description:

| Device                          | Identifier | Description |
| ------------------------------- | ---------- | ----------- |
| `vlan01` (Parent: em1, Tag: 10) | `opt1`     | Trusted     |
| `vlan02` (Parent: em1, Tag: 20) | `opt2`     | IoT         |
| `vlan03` (Parent: em1, Tag: 30) | `opt3`     | Guest       |
|                                 |            |             |

**Save**, then **Apply**.

Expected final assignment table:

|Identifier|Description|Type|Device|
|---|---|---|---|
|`wan`|WAN|hardware|`em0`|
|`lan`|LAN|hardware|`em1`|
|`opt1`|Trusted|vlan|vlan01 (Parent: em1, Tag: 10)|
|`opt2`|IoT|vlan|vlan02 (Parent: em1, Tag: 20)|
|`opt3`|Guest|vlan|vlan03 (Parent: em1, Tag: 30)|

The left navigation now lists interfaces by **Description**, not identifier — look for _Trusted_, _IoT_, _Guest_, not `opt1`.

![[Pasted image 20260920175436.png]]

---

### 4.3 Enable and address each interface

Assignment alone gives an interface no IP and leaves it administratively down. Each one must be configured individually.

**Interfaces → Trusted**, then repeat for IoT and Guest:

|Field|Value|
|---|---|
|Enable Interface|**ticked**|
|IPv4 Configuration Type|**Static IPv4**|
|IPv4 address|see table below, prefix **/24**|
|Block private networks|**unticked**|
|Block bogon networks|**unticked**|

|Interface|Address|
|---|---|
|Trusted|`10.10.10.1` / 24|
|IoT|`10.10.20.1` / 24|
|Guest|`10.10.30.1` / 24|

**Save**, then **Apply changes** in the banner at the top. OPNsense stages changes rather than applying them immediately.

> **Leave both block checkboxes unticked.** They are WAN-side options. Ticking them on an internal VLAN silently drops all RFC1918 traffic, which looks exactly like a routing failure and is tedious to trace.

![[Pasted image 20260920175530.png]]
![[Pasted image 20260920175556.png]]
![[Pasted image 20260920175613.png]]

---

### 4.4 Verify

From the OPNsense console sign-in if need to and select option **8) Shell**:

```
ifconfig vlan01 | grep inet
ifconfig vlan02 | grep inet
ifconfig vlan03 | grep inet
```

Empty output means the interface page in 4.3 has not been completed for that device.

---

### What the output tells you

**All three VLANs share one MAC address** (`00:0c:29:99:2d:5d`, the same as `em1`). This is correct and is the core of the design — one physical NIC, one MAC, three logical networks distinguished only by a 4-byte tag inserted into each frame.

**MTU stays 1500 on the VLAN devices.** The 802.1Q tag adds 4 bytes on the wire, so tagged frames reach 1522 bytes rather than 1518. Cheap switches that drop these are the origin of the "baby giant" frame category.

**`status: active`** confirms the parent link is up. A VLAN device on a dead parent shows `no carrier` regardless of its own configuration.

### Current state after Part 4

| Interface | Device   | Tag      | Address          | Role                 |
| --------- | -------- | -------- | ---------------- | -------------------- |
| WAN       | `em0`    | —        | DHCP from VMnet8 | Internet uplink      |
| LAN       | `em1`    | untagged | `10.10.99.1/24`  | Management back door |
| Trusted   | `vlan01` | 10       | `10.10.10.1/24`  | Full access          |
| IoT       | `vlan02` | 20       | `10.10.20.1/24`  | Internet only        |
| Guest     | `vlan03` | 30       | `10.10.30.1/24`  | Internet only        |

### Next: DHCP and firewall rules

Do **not** tag any clients yet. The VLANs have no DHCP servers, so a tagged client gets no address and appears broken when nothing is wrong.

1. **Services → DHCPv4 → [each VLAN]** — enable, range `.100`–`.199`
2. **Firewall → Aliases** — create `RFC1918` network alias
3. **Firewall → Rules → [each VLAN]** — block rules _above_ the permissive pass rule

## Part 5: Setup DHCP per VLAN Interface (Kea)

This part involves DHCP for all three VLAN VLAN interface (not client). One scope per VLAN, serving whatever devices tag into it.

>  OPNsense 26.7 uses **Kea DHCP** (`Services → Kea DHCP → Kea DHCPv4`). 
### Kea vs dnsmasq
Both hand out addresses. They differ in what else they do and how much control they give you.

**dnsmasq** is DNS + DHCP service written for small networks — a home router, a lab, a single subnet. 
- Its strength is that the two services know about each other: a client that gets a DHCP lease automatically becomes resolvable by hostname. Configuration is minimal because it infers most things.

**Kea** is a purpose-built DHCP server from ISC.
- It does DHCP only, and is designed for networks with many subnets, high lease volume, failover between two servers, and external database backends. 

|                       | dnsmasq                                        | Kea                               |
| --------------------- | ---------------------------------------------- | --------------------------------- |
| **Socket binding**    | One wildcard on `*:67`                         | One per interface address         |
| **Scope selection**   | Matches the receiving interface after the fact | Socket per subnet decides it      |
| **Config model**      | Ranges, mostly inferred                        | Explicit subnets in CIDR          |
| **DNS included**      | Yes                                            | No                                |
| **High availability** | No                                             | Yes, active/active peers          |
| **Reservations**      | Simple host entries                            | Per-subnet, with classes and tags |
| **Intended scale**    | Single subnet, small networks                  | Many subnets, enterprise          |

---

### 5.1 Why each VLAN needs its own scope

The address a client receives is what makes VLAN membership mean something at layer 3.

A client tags its DHCP discover with VLAN 20. That frame reaches `em1`, OPNsense strips the tag and hands it to `vlan02` — the only interface receiving tag 20 traffic. The DHCP scope bound to that interface answers.

The client gets `10.10.20.x` with gateway `10.10.20.1`. It could not have received a `10.10.10.x` address, because the request never touched the Trusted interface.

**The gateway assignment is where isolation gets teeth.** A client in VLAN 20 has exactly one way off its subnet: `10.10.20.1`, which is the firewall. Every packet bound for another VLAN must pass through the rule set. There is no alternate path.

Tagging without separate subnets buys nothing — all clients would share one broadcast domain and talk directly at layer 2, with the firewall never seeing the traffic.

---

### 5.2 Subnets tab

**Services → Kea DHCP → Kea DHCPv4 → Subnets** → **+** for each:

| Subnet          | Description | Pools                       |
| --------------- | ----------- | --------------------------- |
| `10.10.99.0/24` | Management  | `10.10.99.100-10.10.99.199` |
| `10.10.10.0/24` | Trusted     | `10.10.10.100-10.10.10.199` |
| `10.10.20.0/24` | IoT         | `10.10.20.100-10.10.20.199` |
| `10.10.30.0/24` | Guest       | `10.10.30.100-10.10.30.199` |

Save each, then **Apply**.

The management subnet matters: once Kea takes over DHCP, the untagged LAN needs a scope too or clients lose addressing on the back-door network.

### Every field in the subnet dialog

| Field                        | Fill in          | What it does                                                                                                                         |
| ---------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Subnet**                   | **Required**     | Network this scope serves. Kea matches it against interface addresses to pick the right scope.                                       |
| **Description**              | Optional         | Label for your reference, not parsed. Worth setting — the subnet list is unreadable without it.                                      |
| **Pools**                    | **Required**     | Addresses Kea may hand out. Everything outside the range stays free for static assignment.                                           |
| **Valid lifetime**           | No               | Lease duration in seconds. Blank uses the global default (4000s).                                                                    |
| **Match client-id**          | Leave ticked     | Kea identifies clients by DHCP client-identifier rather than MAC. Untick only if a device sends inconsistent client-ids.             |
| **Auto collect option data** | **Leave ticked** | Derives routers, DNS and NTP from the interface's own config. This is what gives VLAN 10 clients gateway `10.10.10.1` automatically. |
| **Static routes**            | No               | Extra routes pushed into clients' routing tables. The default gateway covers everything here.                                        |
| **Classless static routes**  | No               | Same idea, option 121 format. For overriding the default route on complex networks.                                                  |
| **Domain name**              | No               | Suffix given to clients. Cosmetic.                                                                                                   |
| **Domain search**            | No               | Suffix list for resolving short names. Cosmetic.                                                                                     |
| **Time servers**             | No               | NTP. Auto collect already supplies the interface address.                                                                            |
| **Next server**              | No               | PXE boot server address.                                                                                                             |
| **TFTP server**              | No               | PXE — where boot files live.                                                                                                         |
| **TFTP bootfile name**       | No               | PXE — which file to fetch.                                                                                                           |
| **Options**                  | No               | Custom DHCP options defined on the Options tab first.                                                                                |
| **Dynamic DNS**              | No               | Registers client hostnames into DNS. Adds complexity without teaching anything about VLANs.                                          |

Three fields matter. Two checkboxes stay at defaults. Everything else stays blank.

---

### 5.3 Settings tab

Subnets alone do nothing. The service has to be enabled and bound.

- **Settings** → **Enable** — ticked
- **Interfaces** — select `lan`, `opt1`/`Trusted`, `opt2`/`IoT`, `opt3`/`Guest`

Save, then **Apply**.

---

## 5.4 Verify kea-dhcp4 For Each VLAN

Verification came kea setup for each VLAN:

```
sockstat -4 -l | grep :67
```
- Should out put something like:
	![[Pasted image 20260920203243.png]]
- If the output is something like:  `nobody  dnsmasq  64853  4 udp4  *:67  *:*`
	- **Cause:** dnsmasq was already serving DHCP — enabled when the LAN address was set from the console. Two DHCP servers cannot share port 67, so Kea silently failed to start.
	- **Fix:** **Services → Dnsmasq DNS & DHCP → DHCP ranges** — delete every entry. dnsmasq serves DHCP whenever it has ranges; with none defined it stops.
		- **Do not untick Enable on the dnsmasq General tab.** That kills the service entirely including DNS.
	- Then reset `kee` service on OPNsense: `service kea restart`

---
### 5.5 Lease check

On each client VM run:

```
udhcpc -i eth0
ip -4 addr show eth0
```
- Device has requested an address on them


Then **Services → Kea DHCP → Leases DHCPv4** in portal.
- The table becomes the main verification tool for tagging. 
- Once a client tags into VLAN 10, a `10.10.10.x` lease appears under a Trusted group, proving the tag reached the right interface. Cleaner evidence than any command run on the client.

---

### Current state after Part 5

| Interface        | Address         | DHCP pool     | Leases                    |
| ---------------- | --------------- | ------------- | ------------------------- |
| LAN (untagged)   | `10.10.99.1/24` | `.100`–`.199` | client1, client2, client3 |
| Trusted (tag 10) | `10.10.10.1/24` | `.100`–`.199` | none yet                  |
| IoT (tag 20)     | `10.10.20.1/24` | `.100`–`.199` | none yet                  |
| Guest (tag 30)   | `10.10.30.1/24` | `.100`–`.199` | none yet                  |

---

### Next: firewall rules

Every VLAN interface currently has zero firewall rules
- Using default deny.

---


## Part 6: Firewall rules

### 6.1 The rule that matters most

A freshly created OPNsense interface has no rules, and no rules means deny everything.

---
## 6.2 Alias

Add: **Firewall → Aliases** → **+**

| Field   | Value                                           |
| ------- | ----------------------------------------------- |
| Name    | `RFC1918`                                       |
| Type    | Network(s)                                      |
| Content | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
- Those three ranges are the entire private address space, reserved for internal use and never routable on the public internet:
- `RFC1918` stops all lateral movement at once — Guest cannot reach Trusted, IoT, the management network, or the firewall's own interfaces.

|Range|Size|Where you see it|
|---|---|---|
|`10.0.0.0/8`|16.7M addresses|This whole lab|
|`172.16.0.0/12`|1M addresses|Docker default, some corporate networks|
|`192.168.0.0/16`|65K addresses|Home routers, VMware NAT|

---

## 6.4 Trusted

Add one rule: **Firewall → Rules → Trusted** → **+**

| Field       | Value                |
| ----------- | -------------------- |
| Action      | Pass                 |
| Interface   | Trusted              |
| Direction   | in                   |
| Protocol    | any                  |
| Source      | **Trusted net**      |
| Destination | any                  |
| Description | Trusted: full access |

Trusted can reach everything - internet, other VLANs, the firewall's web UI.

---

## 6.5 Guest

**Firewall → Rules → Guest +**

| #   | Action | Protocol | Source    | Destination   | Port | Description         |
| --- | ------ | -------- | --------- | ------------- | ---- | ------------------- |
| 1   | Pass   | UDP      | Guest net | Guest address | 53   | DNS                 |
| 2   | Pass   | UDP      | Guest net | Guest address | 67   | DHCP                |
| 3   | Block  | any      | Guest net | `RFC1918`     | —    | No lateral movement |
| 4   | Pass   | any      | Guest net | any           | —    | Internet            |

---

## 6.6 IoT

**Firewall → Rules → Guest +**

| #   | Action | Protocol | Source  | Destination | Port | Description         |
| --- | ------ | -------- | ------- | ----------- | ---- | ------------------- |
| 1   | Pass   | UDP      | IoT net | IoT address | 53   | DNS                 |
| 2   | Pass   | UDP      | IoT net | IoT address | 67   | DHCP                |
| 3   | Block  | any      | IoT net | `RFC1918`   | —    | No lateral movement |
| 4   | Pass   | any      | IoT net | any         | —    | Internet            |

---

## 6.7 Expected behaviour

|From|To|Expect|Why|
|---|---|---|---|
|Trusted|its gateway|Reply|Pass-any rule|
|Trusted|internet|Reply|Pass-any rule|
|Trusted|Guest client|Reply|Trusted may cross|
|Guest|its gateway|Reply|DNS/DHCP carve-outs|
|Guest|internet|Reply|Rule 4|
|Guest|Trusted client|**Timeout**|Rule 3|
|Guest|IoT client|**Timeout**|Rule 3|
|IoT|anything|**Timeout**|No rules, default deny|

---

## Part 7: Tagging clients and testing isolation

Last part: tag each client into its VLAN, then run the tests to see if fireewall rules worked

---

### 7.1 Tag the clients

On **client1** (VLAN 10):

```
ip link set eth0 up
ip link add link eth0 name eth0.10 type vlan id 10
ip link set eth0.10 up
udhcpc -i eth0.10
```

On **client2** (VLAN 20): substitute `eth0.20` and `id 20`.

On **client3** (VLAN 30): substitute `eth0.30` and `id 30`.

#### Make it survive reboot

Add to `/etc/network/interfaces`:

```
auto eth0
iface eth0 inet manual

auto eth0.10
iface eth0.10 inet dhcp
    vlan-raw-device eth0
```

Requires `apk add vlan`, installed earlier.

---

## 7.2 Confirm tagging worked

On each client:

```
ip -4 addr show eth0.10
ip route
```

Expect `10.10.10.x` and a default route via `10.10.10.1`.

---

## 7.3 Tests

### From client3 — Guest (four rules)

```
ping -c3 10.10.30.1        # gateway
ping -c3 1.1.1.1           # internet by IP
ping -c3 google.com        # internet by name (tests DNS)
ping -c3 10.10.10.100      # Trusted client
ping -c3 10.10.20.100      # IoT client
ping -c3 10.10.99.1        # management interface
```

| Target         | Expect      | Why                                        |
| -------------- | ----------- | ------------------------------------------ |
| `10.10.30.1`   | Reply       | DNS/DHCP carve-outs, plus ICMP via rule 4  |
| `1.1.1.1`      | Reply       | Rule 4 — internet is outside RFC1918       |
| `google.com`   | Reply       | Proves DNS resolution works                |
| `10.10.10.100` | **Timeout** | Rule 3 blocks RFC1918                      |
| `10.10.20.100` | **Timeout** | Rule 3                                     |
| `10.10.99.1`   | **Timeout** | Rule 3 — management is inside `10.0.0.0/8` |

### From client1 — Trusted (one pass rule)

```
ping -c3 10.10.10.1        # gateway
ping -c3 1.1.1.1           # internet
ping -c3 10.10.30.100      # Guest client
ping -c3 10.10.20.100      # IoT client
```

All four should reply for client1.

### From client2 — IoT (no rules)

```
ping -c3 10.10.20.1        # gateway
ping -c3 1.1.1.1           # internet
```

Both should **time out**. Everything fails — that is default-deny.


---


## 7.4 Firewall Log

In Web, open **Firewall → Log Files → Live View** in the web UI while running the tests. Filter by source address.

Every blocked packet appears immediately with the rule that dropped it.


![[Pasted image 20260920212810.png|549]]

---
---

## Verification complete

|Layer|Proven by|
|---|---|
|802.1Q tagging|Three distinct lease groups in Kea|
|Per-VLAN routing|Each client reaching its own gateway|
|Default deny|IoT with no rules reaching nothing|
|Rule enforcement|Guest timeouts with matching log entries|
|Evaluation direction|Trusted → Guest works, Guest → Trusted does not|
