# Containerlab FRR MPLS L3VPN Troubleshooting Log

This document summarizes all deployment, Docker volume mount, Linux kernel MPLS, and FRR 9.1 configuration issues resolved while building the **MPLS L3VPN lab (`clab-mpls`)** in Containerlab.

---

## 1. Containerlab Topology & Docker Bind Mount Issues

### 1.1 Relative Destination Mount Path on `ce2b`
* **Symptom:**
  ```text
  ERRO node "ce2b" deploy: invalid mount path: 'etc/frr/frr.conf' mount path must be absolute
  ```
* **Root Cause:** Docker bind mounts require an absolute container destination path starting with `/`.
* **Fix:** Added the leading slash (`/etc/frr/frr.conf`) in `.clab.yml`.

### 1.2 Wrong Configuration Destination Path on CE Nodes
* **Symptom:** `ce1a` exited or booted without loading `ce1a.conf` (`/etc/frr/frr.conf does not exist; skipping config apply`).
* **Root Cause:** `ce1a.conf`, `ce2a.conf`, and `ce1b.conf` were mounted to `/etc/frr.conf` instead of `/etc/frr/frr.conf`.
* **Fix:** Updated all CE bind mounts to target `/etc/frr/frr.conf` and recreated containers using `sudo containerlab deploy -t <topology>.clab.yml --reconfigure`.

### 1.3 Missing `-vrf.conf` Mounts and Wrong `daemons` File on PE Nodes
* **Symptom:** `bgpd is not running` on `pe1` and `pe2`, and `Unable to create lock file: /etc/frr/pe2-vrf.conf, No such file or directory`.
* **Root Cause:** `pe2` was mounting the generic `daemons` file (where `bgpd=no`) instead of `daemons-pe` (`bgpd=yes`, `ldpd=yes`, `ospfd=yes`), and `pe1-vrf.conf` / `pe2-vrf.conf` were not bound into the containers.
* **Fix:** Updated `pe1` and `pe2` binds in `.clab.yml`:
  ```yaml
  binds:
    - daemons-pe:/etc/frr/daemons
    - configs/pe2.conf:/etc/frr/frr.conf
    - configs/pe2-vrf.conf:/etc/frr/pe2-vrf.conf
  ```

### 1.4 YAML List Formatting & `ip link` Command Typos
* **Symptom:**
  * `yaml: unmarshal errors: line 67: cannot unmarshal !!str '-ip lin...' into []string`
  * `failed to execute cmd: "ip link add custB uo"`
* **Root Cause:** Missing space after the YAML list hyphen (`-ip link...` instead of `- ip link...`) and a command typo (`add custB uo` instead of `set custB up`).
* **Fix:** Added a space after every `-` under `exec:` and corrected the command to `ip link set custB up`.

---

## 2. Linux Kernel MPLS Prerequisites

### 2.1 MPLS Kernel Modules & Sysctls
* **Symptom:** `ldpd` failed to activate `address-family ipv4` or populate transport labels in `show mpls table`.
* **Root Cause:** Containerlab containers share the host Linux kernel; MPLS label switching requires kernel modules loaded on the host and MPLS sysctls enabled inside the `pe1`, `pe2`, and `p` containers.
* **Fix:**
  1. Loaded MPLS modules on the Linux host:
     ```bash
     sudo modprobe mpls_router
     sudo modprobe mpls_iptunnel
     ```
  2. Enabled MPLS label space and interface input inside PE and P nodes:
     ```bash
     sysctl -w net.mpls.platform_labels=1048575
     sysctl -w net.mpls.conf.eth1.input=1
     sysctl -w net.mpls.conf.lo.input=1
     ```

---

## 3. FRR 9.1 Configuration & VRF Boot-Timing Fixes

### 3.1 Interface VRF Typo (`eth3vrf`)
* **Symptom:** `line 6: % Unknown command: interface eth3vrf custB`
* **Root Cause:** Missing space between `eth3` and `vrf`.
* **Fix:** Corrected to `interface eth3 vrf custB`.

### 3.2 VRF Creation Order vs. `frr.conf` Startup
* **Symptom:** `Configuration file[/etc/frr/frr.conf] processing failure: 11` when VRF interfaces or VRF BGP instances were placed inside `frr.conf`.
* **Root Cause:** FRR reads `/etc/frr/frr.conf` immediately on container boot—*before* Containerlab executes `ip link add custA type vrf ...` in the `exec:` block.
* **Fix:** Kept only global IGP, LDP, and core VPNv4 iBGP settings in `pe1.conf` / `pe2.conf`, and moved VRF interfaces and `router bgp 65000 vrf <VRF>` blocks into `pe1-vrf.conf` / `pe2-vrf.conf` so they load after the Linux VRFs are created.

### 3.3 FRR 9.1 BGP Address-Family & RD/RT Syntax
* **Symptom:**
  * `line 19: % Unknown command: address-family ipv4`
  * `line 24: % Unknown command: rd 65010:10`
* **Root Cause:**
  * FRR 9.1 requires an explicit sub-type (`address-family ipv4 unicast` for PE-CE VRF peering, or `address-family ipv4 vpn` for PE-PE core peering).
  * Route Distinguishers and Route Targets must be placed inside `address-family ipv4 unicast` under the VRF BGP instance using `rd vpn export` and `rt vpn import/export`.
* **Fix:**
  ```text
  router bgp 65000 vrf custA
   bgp router-id 3.3.3.3
   no bgp ebgp-requires-policy
   neighbor 10.0.0.18 remote-as 65015
   address-family ipv4 unicast
    neighbor 10.0.0.18 activate
    rd vpn export 65015:10
    rt vpn import 65015:10
    rt vpn export 65015:10
    label vpn export auto
    export vpn
    import vpn
   exit-address-family
  !
  ```

### 3.4 `pe2-vrf.conf` Typos (Commas, Duplicate VRF Name, Missing `remote-as`)
* **Symptom:** VPNv4 neighbour failed to activate and `custB` BGP peering failed to load.
* **Root Cause:**
  1. `neighbor 2,2,2,2 activate` used commas instead of periods (`2.2.2.2`).
  2. The second VRF BGP block repeated `router bgp 65000 vrf custA` instead of `vrf custB`.
  3. The `custB` block activated `neighbor 10.0.0.22` without defining `neighbor 10.0.0.22 remote-as <ASN>`.
* **Fix:** Replaced commas with dots, renamed the second VRF block to `vrf custB`, and added `neighbor 10.0.0.22 remote-as <ASN>`.

### 3.5 MPLS LDP Address-Family Syntax & Hidden Whitespace
* **Symptom:** `line 19: % Unknown command: address-family ipv4` inside `mpls ldp`.
* **Root Cause:** Unlike BGP, `mpls ldp` uses `address-family ipv4` (without `unicast`), closes with `exit-address-family`, and rejects invisible non-breaking spaces (`\xC2\xA0`) copied from external editors.
* **Fix:** Stripped non-breaking spaces across all configs (`sed -i 's/\xc2\xa0/ /g' configs/*.conf`) and formatted `mpls ldp` cleanly:
  ```text
  mpls ldp
   router-id 3.3.3.3
   address-family ipv4
    discovery transport-address 3.3.3.3
    interface eth1
   exit-address-family
  !
  ```

---

## 4. Verification Commands

Use these commands to verify control-plane and data-plane health across the lab:

```bash
# Check active FRR daemons on PE routers
sudo docker exec -it clab-mpls-pe1 ps aux | grep -E "zebra|ospfd|bgpd|ldpd"

# Verify OSPF, MPLS LDP, and MPLS label table
sudo docker exec -it clab-mpls-pe1 vtysh -c "show ip ospf neighbor"
sudo docker exec -it clab-mpls-pe1 vtysh -c "show mpls ldp neighbor"
sudo docker exec -it clab-mpls-pe1 vtysh -c "show mpls table"

# Verify VPNv4 core BGP and VRF PE-CE BGP sessions
sudo docker exec -it clab-mpls-pe1 vtysh -c "show bgp ipv4 vpn summary"
sudo docker exec -it clab-mpls-pe1 vtysh -c "show bgp vrf custA summary"
sudo docker exec -it clab-mpls-pe1 vtysh -c "show bgp vrf custB summary"
```
