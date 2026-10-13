 VPC - isolated network in AWS

1. What does a VPC contain?
A VPC is associated with several networking components:

Component	                Purpose
CIDR block	                Defines the VPC's IP address range
Subnets	                    Divide the VPC into smaller networks
Route tables	            Decide where network traffic goes
Internet Gateway (IGW)	    Enables internet connectivity for eligible resources
NAT Gateway	                Allows private-subnet resources to initiate outbound IPv4 internet connections
Security Groups	            Control traffic at the resource's network interface
Network ACLs (NACLs)	    Control traffic at the subnet boundary
VPC endpoints	            Allow private connectivity to supported AWS services

2. How many IP addresses are available in a subnet?
AWS reserves 5 IPv4 addresses in each subnet.
For example, a /24 subnet contains 256 IPv4 addresses, of which 251 are available for allocation by AWS resources.

3. What is an Internet Gateway?
    An Internet Gateway (IGW) is a VPC component that enables communication between a VPC and the internet when routing, addressing, and security rules permit it.
    To make an EC2 instance publicly reachable over IPv4, you generally need:
    1. An Internet Gateway attached to the VPC.
    2. A route from the subnet to the Internet Gateway.
    3. A public IPv4 address or Elastic IP on the instance.
    4. Security group and NACL rules allowing the required traffic.
    5. The application listening on the expected port.
    Attaching an IGW alone does not make every resource public.

4. What is a NAT Gateway?
A NAT Gateway lets resources in private subnets initiate outbound IPv4 connections to the internet without allowing unsolicited inbound internet connections to those resources through that NAT path.

5. What exactly is a network interface?
Think about your laptop for a moment.
Your laptop might have:
- A Wi-Fi adapter.
- An Ethernet port.
- An IP address on your home Wi-Fi.
- A MAC address associated with its network adapter.
The network adapter is what lets your laptop send and receive data over a network.
An AWS Elastic Network Interface (ENI) is the virtual equivalent of a network card.
An EC2 instance uses an ENI to connect to a VPC subnet and communicate with other resources.

6. What is a Transit Gateway?
An AWS Transit Gateway is a central networking hub that connects multiple VPCs and external networks.
Imagine your company has separate AWS VPCs for development, staging, and production. Without a central hub, you might need multiple VPC peering connections to make them communicate.
Transit Gateway simplifies this by allowing each VPC to connect to the same hub.

You can attach all four VPCs to a Transit Gateway.
Traffic can flow between them when the Transit Gateway route tables, VPC route tables, and security rules permit it.
Why use a Transit Gateway?
- Connect many VPCs without creating a full mesh of peering connections.
- Connect VPCs across AWS accounts and, where supported, Regions.
- Connect AWS networks to on-premises networks through VPN or Direct Connect attachments.
- Centralize network routing and apply segmentation through Transit Gateway route tables.
Important: Attaching a VPC to a Transit Gateway doesn't automatically make all other attached VPCs reachable. Routing and security still need to be configured.

7. What is a Virtual Private Gateway?
A Virtual Private Gateway (VGW) is a gateway attached to a single VPC that acts as the AWS-side endpoint for a Site-to-Site VPN connection. It can also be used with AWS Direct Connect through a Direct Connect gateway. 

Imagine your company has an office with servers in its own data center, and your AWS VPC hosts an application. You want the application to communicate privately with those office servers.

The setup typically involves:
1. Attach a VGW to the AWS VPC.
2. Configure a Customer Gateway resource in AWS to represent your on-premises VPN device.
3. Create a Site-to-Site VPN connection between the VGW and the customer gateway device.
4. Configure routes so AWS knows to send traffic for 192.168.0.0/16 through the VPN.
5. Configure the on-premises device and its routing to send return traffic back to AWS.
The VPN provides an encrypted connection over the underlying network, commonly the internet.


What is the difference between a VGW and a Customer Gateway?
This is another common interview question.
Component	                    Meaning
Virtual Private Gateway	        The AWS-side VPN endpoint attached to your VPC
Customer Gateway resource	        AWS configuration representing your company's VPN device
Customer gateway device	        The actual router or firewall at your office or data center
Site-to-Site VPN	        The connection with VPN tunnels between the AWS-side endpoint and your customer gateway device

The Customer Gateway resource is not the physical router itself. It is an AWS resource containing information about that device

8. 

Understand the two ways to connect your office to AWS

Option A: Site-to-Site VPN
Connects your office network to AWS using encrypted VPN tunnels over an underlying IP network, commonly the public internet.
- Usually quicker to set up.
- Uses IPsec encryption.
- Depends on the underlying internet connection when deployed over the public internet.

Option B: AWS Direct Connect
Establishes a dedicated network connection from your data center or office to an AWS Direct Connect location, typically through a network provider.
- More consistent network performance.
- Useful for large data transfers and predictable network connectivity.
- Takes more planning and may take longer to provision.

The key distinction: a Site-to-Site VPN is a VPN connection, while Direct Connect is a dedicated network connection. Direct Connect does not automatically encrypt traffic; if encryption is required, you can layer a VPN over Direct Connect where supported.


What exactly is a Direct Connect Gateway?
A Direct Connect Gateway (DXGW) is a logical AWS resource that helps connect your Direct Connect connection to one or more VPCs through a Virtual Private Gateway or Transit Gateway. It is not the physical connection itself. 

Think of the components as three different things:
- Direct Connect: the dedicated network connection.
- Direct Connect Gateway: a logical connector that links that connection to VPC gateways.
- VGW or TGW: the AWS-side gateway that connects to your VPC or central network.

