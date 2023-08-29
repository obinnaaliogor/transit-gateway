AWS Transit Gateway is a service provided by Amazon Web Services (AWS) that simplifies the network architecture for connecting multiple Amazon Virtual Private Clouds (VPCs) and on-premises networks. It acts as a hub that allows you to interconnect these networks and manage communication between them more efficiently.

Here are some key features and benefits of AWS Transit Gateway:

1. **Centralized Hub**: AWS Transit Gateway serves as a centralized hub for connecting multiple VPCs and on-premises networks. This simplifies network architecture by reducing the number of connections required between each VPC and on-premises network.

2. **Simplified Routing**: With Transit Gateway, you can use a single set of static routes or dynamic Border Gateway Protocol (BGP) routes to control traffic between various networks. This eliminates the need for complex peering relationships between individual VPCs.

3. **Network Segmentation**: Transit Gateway allows you to segment your network into multiple route tables. This enables you to control traffic flow and implement different routing policies based on your organization's requirements.

4. **VPN and Direct Connect Integration**: You can connect your on-premises network to Transit Gateway using AWS Direct Connect or VPN connections, providing a secure and private communication channel between your data center and AWS resources.

5. **Scalability**: Transit Gateway is designed to handle large-scale deployments and can accommodate a significant number of VPCs and connections.

6. **Network Inspection and Monitoring**: AWS Transit Gateway provides visibility into network traffic using Amazon VPC Flow Logs. This helps in monitoring and troubleshooting network issues.

7. **Inter-Region Peering**: You can use Transit Gateway to establish peering connections between VPCs in different AWS regions, simplifying cross-region network architecture.

8. **Integration with AWS Organizations**: Transit Gateway can be integrated with AWS Organizations, allowing you to manage network resources centrally across multiple AWS accounts.

Please note that AWS services and features can evolve over time, so I recommend checking the official AWS documentation for the most up-to-date information on AWS Transit Gateway and its capabilities.

Template for the transit Gateway:

1. Understand the importance of transit gateway and the problems it solve when compared to VPC peering connections, VPN Connection features.

2. Create VPC, Subnets, Route Tables and ec2 vms required for transit gateway.

3. Understand and implement transit gateway concepts(attachment, association and propagation)

4. Implementing AWS Resource Manager basics when implementing cross account transit gateway sharing.

Scenario 1. Implimenting transit gateway with default route tables which are auto generated
scenario 2. Implementing transit gateway sharing accross aws accounts to enable connectivity to cross account vpcs.

scenario 3 Implementing transit gateway with custom route tables(control the connectivity between vpcs using TGW Route tables).


Before Transit Gateway:
1. VPC Peering
2. Transit VPC with IPsec
3. VPN Connection per VPC

After Transit Gateway:
Lots of things solved with single transit gateway.

1.Transit gateway is a region level construct.
2. It supports upto 10,000 routes per TGW
3. Supports upto 5000 amazon VPC attachments per TGW
4. Multiple TGW route tables for final routing control
5. 50 Gbps of bandwidth per attachment per az
6. Centralized hub for routing bw amazon vpcs and on-premises to aws


Template for the transit Gateway:

Service | Name Tag | Service Data
vpc       dev-vpc    10.1.0.0/16
          qa-vpc     10.2.0.0/16
		  shrd-vpc   10.3.0.0/16
		  cadev-vpc  10.4.0.0/16
		  
		  Name Tag           vpc     | azs
subnets.  dev-public subnet  dev-vpc   us-east-2a | 10.1.1.0/24
          qa-public subnet   qa-vpc  | us-east-2a | 10.2.1.0/24
		  shrd public subnet shrd-vpc  us-east-2a   10.3.1.0/24
		  cadev-public subnet cadev-vpc us-east-2a. 10.4.1.0/24
		  
		  Name Tag
		  dev igw
internet gateway 
          qa igw
		  shrd igw
		  cadev igw
		  
		  
		  Name Tag.   Destination  Target
		  dev-vpc-rt  0.0.0.0/0    dev-igw
Route Tables
          qa-vpc-rt   0.0.0.0/0.    qa-igw
		  shrd-vpc-rt 0.0.0.0/0.    shrd-igw  
		  cadev-vpc-rt 0.0.0.0/0.   cadev-igw


Summarily, transit gateway is used to simplify connectivity bw vpcs within the same aws account and also cross account vpcs connection.

You must add transit gateway route to all the created vpc route tables.
Each vpc route table must have a route of 10.0.0.0/8 destination, target transit gateway. This means allowing connection from 10.0.0.0/8 via transit gateway.

You must create transit gateway attachment for each vpc and add/associate the subnet of that vpc to the attachment.
The transit gateway attachment is the relationship bw the tgw, a specific vpc and its subnet.

Observation: Created a second subnet in dev-vpc, this subnet was created in a diff az that was not associated to the tgw attachment. 
Then i created an ec2 instance in this subnet (dev-public subnet az2--us-east-1b).
From the created ec2 instance, i telnet into other vpc example

sudo yum install telnet -y

telnet <qa-ec2-ip-address> 22
This test failed.

Reason:
This was b/c the ec2 instance was created in a subnet that was not added to the transit gateway attachment.
You must add all subnet within a vpc inthe tgw attachment.

Adding the newly created subnet in the tgw attachment solved the issue.
I was able to telnet back and fort the various vpcs.


CROSS ACCOUNT CONNECTIVITY OF VPC USING TRANSIT GATEWAY AND AWS RESOURCE ACCESS MANAGER.
scenario 2. Implementing transit gateway sharing accross aws accounts to enable connectivity to cross account vpcs.

1. We have a create a vpc and its component in this new aws account following the same blueprint as our previous vpcs.

2. Create aws resource access manager in the primary account.
This is used to Share AWS resources with other accounts or AWS Organizations.

We will share the tgw created in our primary account with our secondary aws account or second aws account. This way we do not need to create another tgw in the second account.

In creating the resource access manager, enter the principle <aws account id> that the transit gateway should be shared to.

Once this is created a request will be sent to that aws secondary account for you to accept the invitation, once the request is  accepted you have completely shared the transit gateway created in the primary account with this secondary account.

3. Now create a tgw attachment for your cadev-vpc. In the secondary account.
  say cadev-tgw-attachment
  . attachment type: VPC
    Enable dns support
  .  VPC ID: cadev-vpc id
  . subnetIDS: ids of subnets created in the cadev vpc
  
  Important once you create the tgw attachment in the secondary account, a request will be sent back to the primary aws account for your acceptance or approval of the creation. This request will be seen when you check the transit gateway attachment in the primary account. click on actions and accept it. 
  once accepted, the status of the tgw attachment will be available after a few min.
  
  Observation:
  Click on transit gateway in the secondary account, you will see the transit gateway same as the one in the primary account
  click on transit gateway attachment in the secondary account, youll see only the cadev-tgw attachment that was created in the secondary account.
  
Get to your primary account and click on transit gateway attachment, youll be able to see both the cadev-tgw attachment and all the other tgw attachment for other vpcs.


4. Luanch ec2 instance in the secondary aws account in the cadev-subnet and try a telnet connection to instances running in the subnets of the vpcs in your primary account.

Create cadev-security group and open ssh port 22 from anywhere.
sudo yum install telnet -y

Test connection:
telnet 10.1.2.61 22
Trying 10.1.2.61...

connection will fail,

REason: we did not add transit gateway route to the cadev-vpc-rt.
We should edit the route table of the cadev vpc and add route of 10.0.0.0/8 via transit gateway this will allow connection via tgw.

test again:

telnet 10.1.2.61 22
Trying 10.1.2.61...
Connected to 10.1.2.61.
Escape character is '^]'.
SSH-2.0-OpenSSH_8.7

Now connection has been established.
Try this from the instance in the primary account to the instances in the secondary account and vice versa. This shows successfull connectivity of the vpcs using transit gateway, aws resource access manager.

  
 	



