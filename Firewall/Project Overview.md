The primary objective of this project is to replace the home lab's permissive inter-VLAN routing with a strict, least-privilege ACL architecture. Instead of every VLAN reaching every other VLAN by default, each segment gets an explicit ruleset. If a connection isn't allow-listed, it doesn't happen. That's the zero-trust model applied directly to VLAN segmentation.

## Core Security Principles
*   **Default-Deny Inter-VLAN Routing:** All traffic between VLANs is blocked by default. Nothing crosses without an explicit rule allowing it.
*   **Strict DMZ Isolation (VLAN 40):** Containing a compromised internet-facing service is the top priority. DMZ hosts cannot initiate connections into Management, Core, Apps, Lab, or Trusted Client segments.
*   **Management Plane Protection (VLAN 10):** The management network is cut off from general traffic. Only designated administrative access can reach it.
*   **Explicit Allow-Listing:** Inter-VLAN traffic that has to happen. Trusted Clients hitting Core for DNS, Management reaching other subnets to administer them is allow-listed rule by rule, not opened broadly.

## Technical Methodology & Rule Design
The rules are built iteratively around one fact about pfSense: it filters statefully on ingress.
*   **Ingress-Based Rule Enforcement:** Rules live on the interface where the packet actually enters the firewall. A rule placed on a destination VLAN's tab never matches traffic sourced from another segment, regardless of the configured source field.
*   **Accurate Traffic Modeling:** In documentation tables (e.g. "Inbound to VLAN X"), rows showing traffic sourced from another internal VLAN are cross-references only. The functional ACL always lives on the originating source VLAN's own configuration.
