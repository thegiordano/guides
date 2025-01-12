## Networking Overview

### VPC
---
#### Basics
A VPC spans all Availability Zones (AZ) in a Region.
Private and Public Subnets can be created per AZ inside a single VPC.

---
#### IGW
- An internet gateway (IGW) enables resources in your public subnets to connect to the internet if the resource has a public IPv4 address or an IPv6 address.
- An IGW provides a target in your VPC route tables for internet-routable traffic. For communication using IPv4, the IGW also performs network address translation (NAT)

---
#### Security Groups
Security Group (Stateful):
- Operates at the instance level.
- Automatically allows response traffic (stateful behavior).
- Rules can only allow traffic, not deny it.
- Default behavior: Deny all inbound and allow all outbound.

Example:
- Inbound rule: Allow HTTP (port 80) from 0.0.0.0/0.
- When a user accesses the server on port 80, the response traffic is automatically allowed.

---
#### Network Access Control Lists
NACL (Stateless):
- Operates at the subnet level.
- Does not allow response traffic automatically; both inbound and outbound rules must be defined.
- Rules can allow or deny traffic.
- Default behavior: Allow all inbound and outbound.
- Rules are evaluated sequentially, starting from the lowest number to the highest.

Example:
- Inbound rule: Allow HTTP (port 80) from 0.0.0.0/0.
- Outbound rule: Must explicitly allow HTTP (port 80) to 0.0.0.0/0, or response traffic will be blocked.

### Advanced services
---
#### VPC Gateway Endpoint
Purpose: Allows private connectivity to S3 and DynamoDB from within your VPC.
Type: Operates at the gateway level.

Targeted Services:
- S3 (Amazon Simple Storage Service)
- DynamoDB

How It Works:
- Adds a route to your route table, directing traffic for the target service (e.g., S3) through the endpoint.
- Does not use private IP addresses within your subnet.
- Traffic remains within the AWS network.

Cost: No additional charge beyond the data transfer charges.

Configuration:
- Simple to set up by creating a gateway endpoint and updating route tables.
- No need for additional network interfaces in your VPC.

Use Case:
- Access S3 or DynamoDB from within your VPC securely and efficiently.

---
#### VPC Interface Endpoint
Purpose: Enables private connectivity to most AWS services and some third-party services via AWS PrivateLink.
Type: Operates at the interface level.

Targeted Services:
- A wide range of AWS services (e.g., EC2, API Gateway, Kinesis, Lambda, etc.).
- Some third-party services available in AWS Marketplace.

How It Works:
- Creates one or more elastic network interfaces (ENIs) in your VPC.
- ENIs use private IP addresses from your subnet.
- Traffic between your VPC and the service is routed through these ENIs.

Cost:
- Charged for each hour that the endpoint is active and for data processing through the endpoint.

Configuration:
- Requires assigning private IP addresses to ENIs in subnets.
- Security groups can be attached to the ENIs for access control.

Use Case:
- Access AWS services that don’t support gateway endpoints.
- Use for more granular control with security groups and private IPs.

---
#### Transit Gateway
Purpose: Simplifies connecting multiple VPCs and on-premises networks using a centralized hub-and-spoke model.
Type: AWS-managed network infrastructure.

Capabilities:
- Facilitates VPC-to-VPC communication (even with overlapping CIDRs).
- Allows on-premises network-to-VPC connectivity.
- Acts as a scalable central hub for routing traffic.

How It Works:
- All connected VPCs and on-premises networks communicate through the transit gateway, avoiding the public internet.
- Route tables within the transit gateway determine the flow of traffic between VPCs and on-premises networks.

Cost:
- Charges based on the number of attachments (per hour) and the amount of data transferred.

Configuration:
- Requires setting up attachments (e.g., VPC attachment, VPN attachment, Direct Connect).
- Configuring route tables to control how traffic flows between connected networks.

Use Cases:
- Large-scale, multi-VPC environments.
- Connecting on-premises networks to multiple VPCs.

---
#### AWS VPC Peering
Purpose: Provides a direct, one-to-one private connection between two VPCs.
Type: Peer-to-peer connection.

Capabilities:
- Enables communication between two VPCs privately.
- VPCs can be in the same AWS region or different regions.

How It Works:
- A peering connection is established between two VPCs.
- Route tables are updated to enable communication between the VPCs.
- No transitive routing: Traffic cannot flow between VPCs indirectly (e.g., A -> B -> C).

Cost:
- No hourly charges for maintaining the connection.
- Data transfer charges apply only for traffic across VPCs in different AZs or Regions.

Configuration:
- Requires creating a peering connection.
- Updating route tables in both VPCs to allow communication.
- No overlap in CIDR blocks is allowed.

Use Cases:
- Direct connection between two specific VPCs.

---
#### Site-to-site VPN
To establish a Site-to-site VPN connection between your pfSense firewall (acting as the customer gateway) and a VPC, do:

**Configure AWS Components**:

Create a Customer Gateway:
- In the AWS Management Console, navigate to the VPC Dashboard.
- Under "Virtual Private Network (VPN)," select "Customer Gateways" and click "Create Customer Gateway."
- Provide a name, select "Static" for routing, and enter the public IP address of your pfSense firewall.
- Click "Create Customer Gateway."

Create a Virtual Private Gateway:
- In the VPC Dashboard, under "Virtual Private Network (VPN)," select "Virtual Private Gateways" and click "Create Virtual Private Gateway."
- Provide a name and use the Amazon default ASN.
- Click "Create Virtual Private Gateway."
- After creation, select the Virtual Private Gateway, choose "Actions," and select "Attach to VPC."
- Select the desired VPC and confirm the attachment.

Create a VPN Connection:
- In the VPC Dashboard, under "Virtual Private Network (VPN)," select "Site-to-Site VPN Connections" and click "Create VPN Connection."
- Provide a name, select the Virtual Private Gateway and Customer Gateway created earlier.
- For routing options, choose "Static" and enter the CIDR block(s) of your on-premises network behind the pfSense firewall.
- Click "Create VPN Connection."
- After creation, select the VPN Connection and download the configuration file, choosing "pfSense" as the vendor and the appropriate software version.

**Configure pfSense**:

Access pfSense Web Interface:
- Open a web browser and navigate to the pfSense management IP address.
- Log in with administrative credentials.

Configure IPsec Tunnels:
- Navigate to "VPN" > "IPsec" and click "Add P1" to create Phase 1 of the tunnel.
- Refer to the downloaded AWS configuration file for specific parameters:
    - Remote Gateway: AWS VPN Endpoint IP address.
    - Authentication Method: Pre-Shared Key (as provided in the AWS config file).
    - Encryption and Hashing Algorithms: Match the AWS configuration.
- Save the Phase 1 configuration.
- Under the newly created Phase 1 entry, click "Show Phase 2 Entries" and then "Add P2" to create Phase 2 of the tunnel.
    - Local Network: Your on-premises subnet(s).
    - Remote Network: AWS VPC CIDR block.
    - Encryption and Hashing Algorithms: Match the AWS configuration.

- Save the Phase 2 configuration and apply changes.

Enable IPsec Service:
- Navigate to "Status" > "IPsec" and ensure the service is running.
- Initiate the connection by clicking "Connect" for the respective tunnel.

Configure Firewall Rules:
- Navigate to "Firewall" > "Rules" and select the "IPsec" tab.
- Create rules to allow traffic between your on-premises network and the AWS VPC as needed.

**Configure AWS Route Tables**:

In order for routes that represent your Site-to-Site VPN connection to automatically appear as propagated routes in your route table, do:

- Select the route table associated with your VPC.
- Edit Route Propagation:
    - With the route table selected, choose "Actions," then "Edit route propagation."
    - In the "Edit route propagation" dialog, select the virtual private gateway (VGW) associated with your VPN connection.
    - Check the box to enable propagation.
    - Click "Save."
