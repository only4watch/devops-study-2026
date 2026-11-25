# Google Cloud Network Connectivity Center (NCC) — Study & Demo Guide
> **Purpose:** Complete study notes, architectural breakdown, conceptual Q&A, and step-by-step lab demo guide for Google Cloud Network Connectivity Center (NCC) and VPC Spokes.
---
## 1. Core Architectural Concepts (Slides 0:00 – 3:44)

Network Connectivity Center (NCC) provides a unified, centralized control plane for managing hybrid and inter-VPC connectivity across Google Cloud.
* **NCC Hub:**
  * A **central, logical global resource** acting as the control-plane manager for your network topology.
  * **Cost:** Free to create (no hourly control-plane charge for the hub resource itself).
  * Does **not** require a dedicated transit VPC or virtual routing appliances; Google manages the underlying data plane at scale.
* **NCC Spokes:**
  * Represent individual network resources attached to a hub across the same or different projects and organizations.
  * **Supported Spoke Types:**
    * **VPC Spokes:** Native VPC networks connected directly for high-scale inter-VPC routing.
    * **Producer VPC Spokes:** Managed service producer networks connected via Private Service Connect (PSC) / peering propagation.
    * **Hybrid Spokes:** Cloud VPN tunnels, Cloud Interconnect VLAN attachments, or Router appliances (SD-WAN) used to connect on-premises/external networks.
    * **NCC Gateway Spokes:** Managed security/inspection checkpoints (e.g., for Hybrid Inspection / SSE topologies).
  * **Cost:** VPC spokes incur an hourly attachment fee plus standard network egress/data-processing charges.
* **Cross-Project & Cross-Organization Governance:**
  * Spokes can reside in different projects or even separate Google Cloud organizations from the Hub.
  * **Hub Admin** manages the hub, topology, and spoke approvals.
  * **Spoke Admin** proposes attaching their VPC or hybrid resource to the hub.

---

## 2. Preset Topologies: Mesh vs. Star (Slides 6:39 – 8:59)

When creating an NCC Hub, you select a **preset topology** that governs how routing domains (spoke groups) exchange routes. **This choice is immutable**—changing topologies requires deleting and recreating the hub.

| Feature / Dimension | Mesh Topology (Default) | Star Topology |
| :--- | :--- | :--- |
| **Spoke Groups** | Single default group (`default`) | Two groups: **Center** (`center`) and **Edge** (`edge`) |
| **Routing Behavior** | **Any-to-Any:** Every spoke exchanges subnet routes with every other spoke attached to the hub. | **Hub-and-Spoke Isolation:** `Center` spokes exchange routes with *all* spokes (`Center` + `Edge`). `Edge` spokes exchange routes *only* with `Center` spokes (never with other `Edge` spokes). |
| **Direct Spoke-to-Spoke** | Allowed across all spokes. | Blocked between `Edge` spokes; allowed between `Center` and `Edge`, and between two `Center` spokes. |
| **Primary Use Case** | Distributed environments where all VPCs/sites require full, direct any-to-any communication. | Enterprise landing zones with shared services/security VPCs in the **Center** and isolated workload/tenant VPCs (Dev, Staging, Prod) at the **Edge**. |
| **Hybrid Spoke Placement** | All hybrid spokes join the single group. | Hybrid spokes with site-to-site data transfer enabled *must* be placed in the **Center** group. |

*(Note: NCC also offers a **Hybrid Inspection** topology with `prod`, `non-prod`, `services`, and `gateways` spoke groups for centralized firewall inspection.)*

---

## 3. NCC VPC Spokes vs. VPC Network Peering (Slides 4:07 – 6:35)

Both technologies keep traffic private on Google's backbone network without traversing the public internet, but NCC solves the operational and scaling limits of traditional peering.

| Comparison Criteria | VPC Network Peering | NCC with VPC Spokes |
| :--- | :--- | :--- |
| **Topology & Scaling** | Point-to-point only. Full mesh requires $\frac{N(N-1)}{2}$ connections (e.g., 10 VPCs = 45 peerings) and hits per-VPC peering limits (default 25). | Centralized Hub-and-Spoke ($O(N)$ attachments). Supports hundreds of VPC spokes per hub. |
| **Transitive Routing** | **Strictly Non-Transitive:** If VPC A peers with B, and B peers with C, A cannot reach C. | **Transitive across the Hub:** Spokes in the same routing domain (or via Center in Star) automatically learn routes and communicate. |
| **Route Control & Filtering** | **All-or-Nothing:** Automatically exchanges all subnet routes; cannot selectively filter which subnets are advertised. | **Granular Export Filters:** Supports **Include** and **Exclude** CIDR export filters per spoke to hide specific subnets from the routing table. |
| **Overlapping Subnets** | **Not Supported:** Peering fails if any subnet ranges overlap between the two VPCs. | **Supported via Export Filters / Private NAT:** Overlapping ranges can be excluded using export filters or translated using Private NAT. |
| **Administrative Overhead** | Requires bilateral configuration (both sides must create peering links for every pair). | Centralized governance with **Auto-Accept project lists** or single-click Hub Admin approval. |
| **Hybrid Connectivity** | Peered VPCs cannot natively share Cloud VPN or Interconnect attachments without custom appliances/BGP workarounds. | Seamlessly integrates VPC spokes with Hybrid spokes (VPN/Interconnect) in one unified control plane. |

---

## 4. Transitive Routing Explained

* **Are VPC spokes transitive?** **Yes — within the rules of the Hub's preset topology.**
  * In a **Mesh** topology, attaching VPC A, VPC B, and VPC C to the same Hub automatically exchanges subnet routes among all three, providing full transitive connectivity without configuring pairwise links.
  * In a **Star** topology, `Edge` spokes are intentionally isolated from one another (`Edge A` cannot talk directly to `Edge B`), while `Center` spokes can reach all `Edge` spokes and hybrid connections.
* **Important Caveat (VPC Peering + NCC):**
  * Transitivity does **not** extend into external VPCs connected via legacy VPC Network Peering. If `VPC A` is an NCC spoke and is also peered to `VPC External` (outside NCC), `VPC External`'s subnet routes are **not** re-advertised into the NCC Hub.

---

## 5. Step-by-Step Lab Demo Guide (Slides 10:11 – 38:51)

Follow these steps to demonstrate NCC setup, route propagation, and subnet filtering live.

### Prerequisites
* At least **2–3 Custom-Mode VPC networks** (Note: *Auto-mode VPCs are not supported as VPC spokes*).
* Non-overlapping primary subnet CIDRs (e.g., `10.10.0.0/24` for `vpc-a`, `10.20.0.0/24` for `vpc-b`), plus one intentionally overlapping or isolated subnet (e.g., `192.168.100.0/24`) to demonstrate export filters.
* At least one test Compute Engine VM per VPC (with ICMP/SSH ingress firewall rules permitted between the internal CIDR ranges).

### Step 1: Create the NCC Hub (10:11)
1. Navigate to **Network Connectivity > Network Connectivity Center** in the Google Cloud Console.
2. Click **Create Hub**.
3. Enter a Hub name (e.g., `ncc-demo-hub`).
4. Under **Preset topology**, select **Mesh** (or **Star** if demonstrating Center/Edge isolation) and click **Create**.

```bash
gcloud network-connectivity hubs create ncc-demo-hub \
    --preset-topology=mesh \
    --description="NCC Demo Hub"
```

### Step 2: Configure Auto-Accept Governance (Optional Cross-Project Demo)
1. Open `ncc-demo-hub` and click **Edit**.
2. Under **Auto-accept VPC spokes**, add trusted **Project IDs**.
   * *Demo Talking Point:* Spokes proposed from listed projects attach immediately without manual intervention; unlisted projects remain in `PENDING` state until approved by the Hub Admin.

### Step 3: Attach VPC Spokes
1. Inside the Hub details page, go to the **Spokes** tab and click **Add Spokes**.
2. Select **VPC network** as the spoke type.
3. Name the spoke (e.g., `spoke-vpc-a`), select `vpc-a`, and repeat for `vpc-b`.
4. *(If cross-project and not auto-accepted)* Switch to the Hub Admin view and click **Accept** on the pending spoke proposal.

```bash
gcloud network-connectivity spokes linked-vpc-network create spoke-vpc-a \
    --hub=ncc-demo-hub \
    --vpc-network=vpc-a \
    --global
```

### Step 4: Verify Dynamic Route Propagation & Connectivity (19:50)
1. Navigate to **VPC Network > VPC networks > `vpc-a` > Routes**.
2. Highlight that `vpc-a` now shows dynamic/NCC-learned routes pointing to `vpc-b`'s subnets (`10.20.0.0/24`).
3. SSH into the VM in `vpc-a` and `ping` the internal IP of the VM in `vpc-b` to confirm private backbone reachability.

### Step 5: Demonstrate Exclude Filters & Overlapping Subnet Handling (22:45 – 38:51)
1. **Scenario A — Isolating a Sensitive Subnet:**
   * Create or attach a spoke (`spoke-vpc-c`) with an **Exclude export range** (e.g., `10.30.99.0/24`).
   * Show in `vpc-a`'s route table that `10.30.0.0/24` is learned, while `10.30.99.0/24` is completely absent from the routing table (enforcing isolation at the routing layer rather than relying solely on firewall rules).
2. **Scenario B — Overlapping Subnet Protection:**
   * Attempt to attach a VPC containing an identical CIDR block already advertised to the hub (`10.10.0.0/24`).
   * Show that NCC **rejects** the spoke attachment to prevent routing ambiguity—and resolve it live by specifying `--exclude-export-ranges=10.10.0.0/24` during spoke creation.

```bash
gcloud network-connectivity spokes linked-vpc-network create spoke-vpc-c \
    --hub=ncc-demo-hub \
    --vpc-network=vpc-c \
    --exclude-export-ranges=10.10.0.0/24 \
    --global
```

---

## 6. Quick Demo Checklist & Gotchas
* **Custom-Mode VPCs Only:** Convert any auto-mode VPC to custom mode before the demo, or spoke creation will fail.
* **Firewall Rules Still Apply:** NCC exchanges *routes*, not firewall permissions. Ensure VPC ingress firewall rules allow traffic from peer spoke CIDRs.
* **No Peering Between Same-Hub Spokes:** Two VPCs attached to the same NCC hub cannot simultaneously maintain a direct VPC Network Peering connection.
* **10-Minute Cooldown:** If you detach a VPC spoke and re-attach it to a *different* hub during a live demo, wait at least 10 minutes for control-plane cleanup.
