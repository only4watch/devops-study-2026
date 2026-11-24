# GCP Network Connectivity Center (NCC) Architectural Guide

The Google Cloud Platform (GCP) Network Connectivity Center (NCC) provides a global integration framework designed to simplify network connectivity across cloud and hybrid enterprise infrastructure. By establishing a central Hub resource, NCC streamlines route distribution and allows organizations to manage complex inter-VPC and hybrid communications without requiring traffic to traverse the public internet or rely on dedicated transit Virtual Private Cloud (VPC) instances.

This guide documents the core architecture, key capabilities, operational workflows, and technical limitations of NCC based on a two-part overview covering Mesh Topology and Star Topology models.

---

## 1. Core Architecture and Concepts

Network Connectivity Center decouples route management from specific network structures by establishing global control planes across GCP resources.

*   **Network Connectivity Center (NCC)**: An overarching integration framework in GCP that unifies and simplifies network connectivity between various spoke resources via a central control point.
*   **Hub**: A global logical resource where spokes are attached. The Hub manages route distribution centrally across connected endpoints and does not require a dedicated VPC instance to function.
*   **Spoke**: Represents Google Cloud network resources attached to the Hub. Spokes can include native VPC networks as well as hybrid connectivity resources such as Cloud VPN and Cloud Interconnect.

---

## 2. Mesh Topology Model

Mesh Topology (covered in Part 1) establishes full-mesh connectivity across connected network resources, eliminating the overhead associated with managing traditional point-to-point network configurations.

### Inter-VPC Connectivity and Route Management

In a Mesh topology, all attached VPC spokes are grouped within a single default group. This arrangement enables direct communication between every attached VPC spoke. Compared to traditional VPC Peering, Mesh topology significantly simplifies inter-VPC networking by managing route exchange holistically rather than through paired peering connections.

The Hub maintains a centralized route table that attached VPCs automatically import. This centralized route exchange provides scalable, secure communication across projects without needing internet transit.

### Practical Demonstrations and Operational Mechanics

Setting up a Mesh deployment involves key management procedures demonstrated in practical workflows:

*   **Hub and Spoke Provisioning**: Creating a central Hub and attaching VPC spokes across single or multi-project environments.
*   **Route Filtering**: Applying exclude export filters to control which specific subnets are advertised to other VPC spokes, ensuring precise route propagation.
*   **Handling IP Range Overlaps**: Addressing configuration errors triggered by overlapping IP ranges across spokes, following remediation procedures to ensure clean route exchange.

### Key Limitations and Cost Considerations

Deployment of Mesh topology requires adhering to specific technical and financial boundaries:

*   **Non-Transitive Connectivity**: Connectivity is non-transitive between NCC spokes and traditional VPC Peering connections.
*   **Unsupported Features**: Static routes, IPv6 dynamic routes, and auto-mode VPC networks are not supported.
*   **Billing Structure**: Creating an NCC Hub is free of charge. However, attached VPC spokes incur hourly charges, which can scale significantly depending on the size of the deployment.

---

## 3. Star Topology Model

Star Topology (covered in Part 2) introduces a hierarchical architecture designed for strict traffic separation, workload isolation, and centralized service delivery.

### Hierarchical Architecture and Connectivity Rules

Unlike full-mesh communication where all VPCs interact freely, Star topology divides spokes into Center and Edge groups with explicit traffic enforcement:

*   **Center-to-Edge & Center-to-Center Communication**: Designated Center spokes can communicate with all Edge spokes and with other Center spokes.
*   **Edge-to-Center Communication**: Edge spokes can communicate exclusively with designated Center spokes.
*   **Edge-to-Edge Blockade**: Communication directly between Edge spokes is explicitly unsupported and blocked by the Hub.

### Enterprise Use Cases

Star topology is ideal for maintaining segmentation and separation of duties across enterprise environments. It serves scenarios where workloads in separate VPCs do not need to interact directly but require access to centralized shared services or Network Virtual Appliances (NVAs).

### Configuration Workflows and Testing

Implementation of Star topology encompasses specific administrative steps:

*   **Group Definition**: Establishing a Star Hub in the GCP console and defining explicit Center and Edge groups.
*   **Spoke Attachment and Verification**: Adding VPC spokes to their respective groups and conducting connectivity validation via Virtual Machine (VM) ping tests. Testing confirms that Edge-to-Center traffic is permitted while Edge-to-Edge traffic is blocked.
*   **Routing Table Audit**: Reviewing NCC routing tables to confirm that active route distribution matches the configured segmentation rules.

### Technical Limitations

Star topology enforces stricter boundary conditions than Mesh topology:

*   **No Hybrid Spokes**: Star topology does not support hybrid connectivity spokes such as Cloud Interconnect or Cloud VPN.
*   **No Dynamic Route Exchange**: Dynamic route exchange across spokes is not supported in Star topology.

---

## 4. Architectural Comparison Matrix

The table below summarizes the structural differences, feature support, and functional trade-offs between Mesh Topology and Star Topology in Network Connectivity Center.

| Feature / Dimension | Mesh Topology (Part 1) | Star Topology (Part 2) |
| :--- | :--- | :--- |
| **Network Structure** | Full-mesh intercommunication | Hierarchical (Center and Edge groups) |
| **Communication Scope** | All spokes communicate via default group | Edge spokes connect only to Center spokes |
| **Edge-to-Edge Traffic** | Allowed | Blocked |
| **Hybrid Spoke Support** | Supported (VPN / Interconnect) | Not Supported |
| **Dynamic Route Exchange** | Supported via central Hub route table | Not Supported |
| **Primary Use Cases** | Simplified multi-VPC connectivity | Workload segmentation & centralized NVAs/shared services |
| **Cost Model** | Free Hub; hourly charge for VPC spokes | Free Hub; hourly charge for VPC spokes |
| **Route Filtering** | Exclude export filters supported | Governed by Center/Edge group rules |

---

## 5. Summary of Limitations

When designing networks with NCC, consider the combined list of architectural constraints across both models:

*   Connectivity between NCC and VPC Peering is non-transitive.
*   Static routes and IPv6 dynamic routes are unsupported.
*   Auto-mode VPC networks cannot be attached as spokes.
*   Overlapping IP address ranges across spokes cause errors requiring explicit remediation.
*   Star topology excludes hybrid spokes (VPN/Interconnect) and dynamic route exchange.
