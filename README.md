AWS VPC Terraform module
Terraform module which creates VPC resources on AWS.

SWUbanner

Usage
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = true

  tags = {
    Terraform = "true"
    Environment = "dev"
  }
}
External NAT Gateway IPs
By default this module will provision new Elastic IPs for the VPC's NAT Gateways. This means that when creating a new VPC, new IPs are allocated, and when that VPC is destroyed those IPs are released. Sometimes it is handy to keep the same IPs even after the VPC is destroyed and re-created. To that end, it is possible to assign existing IPs to the NAT Gateways. This prevents the destruction of the VPC from releasing those IPs, while making it possible that a re-created VPC uses the same IPs.

To achieve this, allocate the IPs outside the VPC module declaration.

resource "aws_eip" "nat" {
  count = 3

  vpc = true
}
Then, pass the allocated IPs as a parameter to this module.

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  # The rest of arguments are omitted for brevity

  enable_nat_gateway  = true
  single_nat_gateway  = false
  reuse_nat_ips       = true                    # <= Skip creation of EIPs for the NAT Gateways
  external_nat_ip_ids = "${aws_eip.nat.*.id}"   # <= IPs specified here as input to the module
}
Note that in the example we allocate 3 IPs because we will be provisioning 3 NAT Gateways (due to single_nat_gateway = false and having 3 subnets). If, on the other hand, single_nat_gateway = true, then aws_eip.nat would only need to allocate 1 IP. Passing the IPs into the module is done by setting two variables reuse_nat_ips = true and external_nat_ip_ids = "${aws_eip.nat.*.id}".

NAT Gateway Scenarios
This module supports three scenarios for creating NAT gateways. Each will be explained in further detail in the corresponding sections.

One NAT Gateway per subnet (default behavior)
enable_nat_gateway = true
single_nat_gateway = false
one_nat_gateway_per_az = false
Single NAT Gateway
enable_nat_gateway = true
single_nat_gateway = true
one_nat_gateway_per_az = false
One NAT Gateway per availability zone
enable_nat_gateway = true
single_nat_gateway = false
one_nat_gateway_per_az = true
If both single_nat_gateway and one_nat_gateway_per_az are set to true, then single_nat_gateway takes precedence.

One NAT Gateway per subnet (default)
By default, the module will determine the number of NAT Gateways to create based on the max() of the private subnet lists (database_subnets, elasticache_subnets, private_subnets, and redshift_subnets). The module does not take into account the number of intra_subnets, since the latter are designed to have no Internet access via NAT Gateway. For example, if your configuration looks like the following:

database_subnets    = ["10.0.21.0/24", "10.0.22.0/24"]
elasticache_subnets = ["10.0.31.0/24", "10.0.32.0/24"]
private_subnets     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24"]
redshift_subnets    = ["10.0.41.0/24", "10.0.42.0/24"]
intra_subnets       = ["10.0.51.0/24", "10.0.52.0/24", "10.0.53.0/24"]
Then 5 NAT Gateways will be created since 5 private subnet CIDR blocks were specified.

Single NAT Gateway
If single_nat_gateway = true, then all private subnets will route their Internet traffic through this single NAT gateway. The NAT gateway will be placed in the first public subnet in your public_subnets block.

One NAT Gateway per availability zone
If one_nat_gateway_per_az = true and single_nat_gateway = false, then the module will place one NAT gateway in each availability zone you specify in var.azs. There are some requirements around using this feature flag:

The variable var.azs must be specified.
The number of public subnet CIDR blocks specified in public_subnets must be greater than or equal to the number of availability zones specified in var.azs. This is to ensure that each NAT Gateway has a dedicated public subnet to deploy to.
"private" versus "intra" subnets
By default, if NAT Gateways are enabled, private subnets will be configured with routes for Internet traffic that point at the NAT Gateways configured by use of the above options.

If you need private subnets that should have no Internet routing (in the sense of RFC1918 Category 1 subnets), intra_subnets should be specified. An example use case is configuration of AWS Lambda functions within a VPC, where AWS Lambda functions only need to pass traffic to internal resources or VPC endpoints for AWS services.

Since AWS Lambda functions allocate Elastic Network Interfaces in proportion to the traffic received (read more), it can be useful to allocate a large private subnet for such allocations, while keeping the traffic they generate entirely internal to the VPC.

You can add additional tags with intra_subnet_tags as with other subnet types.

VPC Flow Log
VPC Flow Log allows to capture IP traffic for a specific network interface (ENI), subnet, or entire VPC. This module supports enabling or disabling VPC Flow Logs for entire VPC. If you need to have VPC Flow Logs for subnet or ENI, you have to manage it outside of this module with aws_flow_log resource.

VPC Flow Log Examples
By default file_format is plain-text. You can also specify parquet to have logs written in Apache Parquet format.

flow_log_file_format = "parquet"
Permissions Boundary
If your organization requires a permissions boundary to be attached to the VPC Flow Log role, make sure that you specify an ARN of the permissions boundary policy as vpc_flow_log_permissions_boundary argument. Read more about required IAM policy for publishing flow logs.

Conditional creation
Prior to Terraform 0.13, you were unable to specify count in a module block. If you wish to toggle the creation of the module's resources in an older (pre 0.13) version of Terraform, you can use the create_vpc argument.

# This VPC will not be created
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  create_vpc = false
  # ... omitted
}
Public access to RDS instances
Sometimes it is handy to have public access to RDS instances (it is not recommended for production) by specifying these arguments:

  create_database_subnet_group           = true
  create_database_subnet_route_table     = true
  create_database_internet_gateway_route = true

  enable_dns_hostnames = true
  enable_dns_support   = true
Network Access Control Lists (ACL or NACL)
This module can manage network ACL and rules. Once VPC is created, AWS creates the default network ACL, which can be controlled using this module (manage_default_network_acl = true).

Also, each type of subnet may have its own network ACL with custom rules per subnet. Eg, set public_dedicated_network_acl = true to use dedicated network ACL for the public subnets; set values of public_inbound_acl_rules and public_outbound_acl_rules to specify all the NACL rules you need to have on public subnets (see variables.tf for default values and structures).

By default, all subnets are associated with the default network ACL.

Public access to Redshift cluster
Sometimes it is handy to have public access to Redshift clusters (for example if you need to access it by Kinesis - VPC endpoint for Kinesis is not yet supported by Redshift) by specifying these arguments:

  enable_public_redshift = true  # <= By default Redshift subnets will be associated with the private route table
Transit Gateway (TGW) integration
It is possible to integrate this VPC module with terraform-aws-transit-gateway module which handles the creation of TGW resources and VPC attachments. See complete example there.

VPC CIDR from AWS IP Address Manager (IPAM)
It is possible to have your VPC CIDR assigned from an AWS IPAM Pool. However, In order to build subnets within this module Terraform must know subnet CIDRs to properly plan the amount of resources to build. Since CIDR is derived by IPAM by calling CreateVpc this is not possible within a module unless cidr is known ahead of time. You can get around this by "previewing" the CIDR and then using that as the subnet values.

Note: Due to race conditions with terraform plan, it is not possible to use ipv4_netmask_length or a pools allocation_default_netmask_length within this module. You must explicitly set the CIDRs for a pool to use.

# Find the pool RAM shared to your account
# Info on RAM sharing pools: https://docs.aws.amazon.com/vpc/latest/ipam/share-pool-ipam.html
data "aws_vpc_ipam_pool" "ipv4_example" {
  filter {
    name   = "description"
    values = ["*mypool*"]
  }

  filter {
    name   = "address-family"
    values = ["ipv4"]
  }
}

# Preview next CIDR from pool
data "aws_vpc_ipam_preview_next_cidr" "previewed_cidr" {
  ipam_pool_id   = data.aws_vpc_ipam_pool.ipv4_example.id
  netmask_length = 24
}

data "aws_region" "current" {}

# Calculate subnet cidrs from previewed IPAM CIDR
locals {
  partition       = cidrsubnets(data.aws_vpc_ipam_preview_next_cidr.previewed_cidr.cidr, 2, 2)
  private_subnets = cidrsubnets(local.partition[0], 2, 2)
  public_subnets  = cidrsubnets(local.partition[1], 2, 2)
  azs             = formatlist("${data.aws_region.current.name}%s", ["a", "b"])
}

module "vpc_cidr_from_ipam" {
  source            = "terraform-aws-modules/vpc/aws"
  name              = "vpc-cidr-from-ipam"
  ipv4_ipam_pool_id = data.aws_vpc_ipam_pool.ipv4_example.id
  azs               = local.azs
  cidr              = data.aws_vpc_ipam_preview_next_cidr.previewed_cidr.cidr
  private_subnets   = local.private_subnets
  public_subnets    = local.public_subnets
}
Examples
Complete VPC with VPC Endpoints.
VPC using IPAM
Dualstack IPv4/IPv6 VPC
IPv6 only subnets/VPC
Manage Default VPC
Network ACL
VPC with Outpost
VPC with secondary CIDR blocks
VPC with unique route tables
Simple VPC
VPC Flow Logs
VPC Block Public Access
Few tests and edge case examples
Contributing
Report issues/questions/feature requests on in the issues section.

Full contributing guidelines are covered here.

Requirements
Name	Version
terraform	>= 1.0
aws	>= 5.79
Providers
Name	Version
aws	>= 5.79
Modules
No modules.

Resources
Name	Type
aws_cloudwatch_log_group.flow_log	resource
aws_customer_gateway.this	resource
aws_db_subnet_group.database	resource
aws_default_network_acl.this	resource
aws_default_route_table.default	resource
aws_default_security_group.this	resource
aws_default_vpc.this	resource
aws_egress_only_internet_gateway.this	resource
aws_eip.nat	resource
aws_elasticache_subnet_group.elasticache	resource
aws_flow_log.this	resource
aws_iam_policy.vpc_flow_log_cloudwatch	resource
aws_iam_role.vpc_flow_log_cloudwatch	resource
aws_iam_role_policy_attachment.vpc_flow_log_cloudwatch	resource
aws_internet_gateway.this	resource
aws_nat_gateway.this	resource
aws_network_acl.database	resource
aws_network_acl.elasticache	resource
aws_network_acl.intra	resource
aws_network_acl.outpost	resource
aws_network_acl.private	resource
aws_network_acl.public	resource
aws_network_acl.redshift	resource
aws_network_acl_rule.database_inbound	resource
aws_network_acl_rule.database_outbound	resource
aws_network_acl_rule.elasticache_inbound	resource
aws_network_acl_rule.elasticache_outbound	resource
aws_network_acl_rule.intra_inbound	resource
aws_network_acl_rule.intra_outbound	resource
aws_network_acl_rule.outpost_inbound	resource
aws_network_acl_rule.outpost_outbound	resource
aws_network_acl_rule.private_inbound	resource
aws_network_acl_rule.private_outbound	resource
aws_network_acl_rule.public_inbound	resource
aws_network_acl_rule.public_outbound	resource
aws_network_acl_rule.redshift_inbound	resource
aws_network_acl_rule.redshift_outbound	resource
aws_redshift_subnet_group.redshift	resource
aws_route.database_dns64_nat_gateway	resource
aws_route.database_internet_gateway	resource
aws_route.database_ipv6_egress	resource
aws_route.database_nat_gateway	resource
aws_route.private_dns64_nat_gateway	resource
aws_route.private_ipv6_egress	resource
aws_route.private_nat_gateway	resource
aws_route.public_internet_gateway	resource
aws_route.public_internet_gateway_ipv6	resource
aws_route_table.database	resource
aws_route_table.elasticache	resource
aws_route_table.intra	resource
aws_route_table.private	resource
aws_route_table.public	resource
aws_route_table.redshift	resource
aws_route_table_association.database	resource
aws_route_table_association.elasticache	resource
aws_route_table_association.intra	resource
aws_route_table_association.outpost	resource
aws_route_table_association.private	resource
aws_route_table_association.public	resource
aws_route_table_association.redshift	resource
aws_route_table_association.redshift_public	resource
aws_subnet.database	resource
aws_subnet.elasticache	resource
aws_subnet.intra	resource
aws_subnet.outpost	resource
aws_subnet.private	resource
aws_subnet.public	resource
aws_subnet.redshift	resource
aws_vpc.this	resource
aws_vpc_block_public_access_exclusion.this	resource
aws_vpc_block_public_access_options.this	resource
aws_vpc_dhcp_options.this	resource
aws_vpc_dhcp_options_association.this	resource
aws_vpc_ipv4_cidr_block_association.this	resource
aws_vpn_gateway.this	resource
aws_vpn_gateway_attachment.this	resource
aws_vpn_gateway_route_propagation.intra	resource
aws_vpn_gateway_route_propagation.private	resource
aws_vpn_gateway_route_propagation.public	resource
aws_caller_identity.current	data source
aws_iam_policy_document.flow_log_cloudwatch_assume_role	data source
aws_iam_policy_document.vpc_flow_log_cloudwatch	data source
aws_partition.current	data source
aws_region.current	data source
Inputs
Name	Description	Type	Default	Required
amazon_side_asn	The Autonomous System Number (ASN) for the Amazon side of the gateway. By default the virtual private gateway is created with the current default Amazon ASN	string	"64512"	no
azs	A list of availability zones names or ids in the region	list(string)	[]	no
cidr	(Optional) The IPv4 CIDR block for the VPC. CIDR can be explicitly set or it can be derived from IPAM using ipv4_netmask_length & ipv4_ipam_pool_id	string	"10.0.0.0/16"	no
create_database_internet_gateway_route	Controls if an internet gateway route for public database access should be created	bool	false	no
create_database_nat_gateway_route	Controls if a nat gateway route should be created to give internet access to the database subnets	bool	false	no
create_database_subnet_group	Controls if database subnet group should be created (n.b. database_subnets must also be set)	bool	true	no
create_database_subnet_route_table	Controls if separate route table for database should be created	bool	false	no
create_egress_only_igw	Controls if an Egress Only Internet Gateway is created and its related routes	bool	true	no
create_elasticache_subnet_group	Controls if elasticache subnet group should be created	bool	true	no
create_elasticache_subnet_route_table	Controls if separate route table for elasticache should be created	bool	false	no
create_flow_log_cloudwatch_iam_role	Whether to create IAM role for VPC Flow Logs	bool	false	no
create_flow_log_cloudwatch_log_group	Whether to create CloudWatch log group for VPC Flow Logs	bool	false	no
create_igw	Controls if an Internet Gateway is created for public subnets and the related routes that connect them	bool	true	no
create_multiple_intra_route_tables	Indicates whether to create a separate route table for each intra subnet. Default: false	bool	false	no
create_multiple_public_route_tables	Indicates whether to create a separate route table for each public subnet. Default: false	bool	false	no
create_private_nat_gateway_route	Controls if a nat gateway route should be created to give internet access to the private subnets	bool	true	no
create_redshift_subnet_group	Controls if redshift subnet group should be created	bool	true	no
create_redshift_subnet_route_table	Controls if separate route table for redshift should be created	bool	false	no
create_vpc	Controls if VPC should be created (it affects almost all resources)	bool	true	no
customer_gateway_tags	Additional tags for the Customer Gateway	map(string)	{}	no
customer_gateways	Maps of Customer Gateway's attributes (BGP ASN and Gateway's Internet-routable external IP address)	map(map(any))	{}	no
customer_owned_ipv4_pool	The customer owned IPv4 address pool. Typically used with the map_customer_owned_ip_on_launch argument. The outpost_arn argument must be specified when configured	string	null	no
database_acl_tags	Additional tags for the database subnets network ACL	map(string)	{}	no
database_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for database subnets	bool	false	no
database_inbound_acl_rules	Database subnets inbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
database_outbound_acl_rules	Database subnets outbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
database_route_table_tags	Additional tags for the database route tables	map(string)	{}	no
database_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
database_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
database_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
database_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
database_subnet_group_name	Name of database subnet group	string	null	no
database_subnet_group_tags	Additional tags for the database subnet group	map(string)	{}	no
database_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
database_subnet_ipv6_prefixes	Assigns IPv6 database subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
database_subnet_names	Explicit values to use in the Name tag on database subnets. If empty, Name tags are generated	list(string)	[]	no
database_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
database_subnet_suffix	Suffix to append to database subnets name	string	"db"	no
database_subnet_tags	Additional tags for the database subnets	map(string)	{}	no
database_subnets	A list of database subnets inside the VPC	list(string)	[]	no
default_network_acl_egress	List of maps of egress rules to set on the Default Network ACL	list(map(string))	
[
  {
    "action": "allow",
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_no": 100,
    "to_port": 0
  },
  {
    "action": "allow",
    "from_port": 0,
    "ipv6_cidr_block": "::/0",
    "protocol": "-1",
    "rule_no": 101,
    "to_port": 0
  }
]
no
default_network_acl_ingress	List of maps of ingress rules to set on the Default Network ACL	list(map(string))	
[
  {
    "action": "allow",
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_no": 100,
    "to_port": 0
  },
  {
    "action": "allow",
    "from_port": 0,
    "ipv6_cidr_block": "::/0",
    "protocol": "-1",
    "rule_no": 101,
    "to_port": 0
  }
]
no
default_network_acl_name	Name to be used on the Default Network ACL	string	null	no
default_network_acl_tags	Additional tags for the Default Network ACL	map(string)	{}	no
default_route_table_name	Name to be used on the default route table	string	null	no
default_route_table_propagating_vgws	List of virtual gateways for propagation	list(string)	[]	no
default_route_table_routes	Configuration block of routes. See https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/default_route_table#route	list(map(string))	[]	no
default_route_table_tags	Additional tags for the default route table	map(string)	{}	no
default_security_group_egress	List of maps of egress rules to set on the default security group	list(map(string))	[]	no
default_security_group_ingress	List of maps of ingress rules to set on the default security group	list(map(string))	[]	no
default_security_group_name	Name to be used on the default security group	string	null	no
default_security_group_tags	Additional tags for the default security group	map(string)	{}	no
default_vpc_enable_dns_hostnames	Should be true to enable DNS hostnames in the Default VPC	bool	true	no
default_vpc_enable_dns_support	Should be true to enable DNS support in the Default VPC	bool	true	no
default_vpc_name	Name to be used on the Default VPC	string	null	no
default_vpc_tags	Additional tags for the Default VPC	map(string)	{}	no
dhcp_options_domain_name	Specifies DNS name for DHCP options set (requires enable_dhcp_options set to true)	string	""	no
dhcp_options_domain_name_servers	Specify a list of DNS server addresses for DHCP options set, default to AWS provided (requires enable_dhcp_options set to true)	list(string)	
[
  "AmazonProvidedDNS"
]
no
dhcp_options_ipv6_address_preferred_lease_time	How frequently, in seconds, a running instance with an IPv6 assigned to it goes through DHCPv6 lease renewal (requires enable_dhcp_options set to true)	number	null	no
dhcp_options_netbios_name_servers	Specify a list of netbios servers for DHCP options set (requires enable_dhcp_options set to true)	list(string)	[]	no
dhcp_options_netbios_node_type	Specify netbios node_type for DHCP options set (requires enable_dhcp_options set to true)	string	""	no
dhcp_options_ntp_servers	Specify a list of NTP servers for DHCP options set (requires enable_dhcp_options set to true)	list(string)	[]	no
dhcp_options_tags	Additional tags for the DHCP option set (requires enable_dhcp_options set to true)	map(string)	{}	no
elasticache_acl_tags	Additional tags for the elasticache subnets network ACL	map(string)	{}	no
elasticache_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for elasticache subnets	bool	false	no
elasticache_inbound_acl_rules	Elasticache subnets inbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
elasticache_outbound_acl_rules	Elasticache subnets outbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
elasticache_route_table_tags	Additional tags for the elasticache route tables	map(string)	{}	no
elasticache_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
elasticache_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
elasticache_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
elasticache_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
elasticache_subnet_group_name	Name of elasticache subnet group	string	null	no
elasticache_subnet_group_tags	Additional tags for the elasticache subnet group	map(string)	{}	no
elasticache_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
elasticache_subnet_ipv6_prefixes	Assigns IPv6 elasticache subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
elasticache_subnet_names	Explicit values to use in the Name tag on elasticache subnets. If empty, Name tags are generated	list(string)	[]	no
elasticache_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
elasticache_subnet_suffix	Suffix to append to elasticache subnets name	string	"elasticache"	no
elasticache_subnet_tags	Additional tags for the elasticache subnets	map(string)	{}	no
elasticache_subnets	A list of elasticache subnets inside the VPC	list(string)	[]	no
enable_dhcp_options	Should be true if you want to specify a DHCP options set with a custom domain name, DNS servers, NTP servers, netbios servers, and/or netbios server type	bool	false	no
enable_dns_hostnames	Should be true to enable DNS hostnames in the VPC	bool	true	no
enable_dns_support	Should be true to enable DNS support in the VPC	bool	true	no
enable_flow_log	Whether or not to enable VPC Flow Logs	bool	false	no
enable_ipv6	Requests an Amazon-provided IPv6 CIDR block with a /56 prefix length for the VPC. You cannot specify the range of IP addresses, or the size of the CIDR block	bool	false	no
enable_nat_gateway	Should be true if you want to provision NAT Gateways for each of your private networks	bool	false	no
enable_network_address_usage_metrics	Determines whether network address usage metrics are enabled for the VPC	bool	null	no
enable_public_redshift	Controls if redshift should have public routing table	bool	false	no
enable_vpn_gateway	Should be true if you want to create a new VPN Gateway resource and attach it to the VPC	bool	false	no
external_nat_ip_ids	List of EIP IDs to be assigned to the NAT Gateways (used in combination with reuse_nat_ips)	list(string)	[]	no
external_nat_ips	List of EIPs to be used for nat_public_ips output (used in combination with reuse_nat_ips and external_nat_ip_ids)	list(string)	[]	no
flow_log_cloudwatch_iam_role_arn	The ARN for the IAM role that's used to post flow logs to a CloudWatch Logs log group. When flow_log_destination_arn is set to ARN of Cloudwatch Logs, this argument needs to be provided	string	""	no
flow_log_cloudwatch_iam_role_conditions	Additional conditions of the CloudWatch role assumption policy	
list(object({
    test     = string
    variable = string
    values   = list(string)
  }))
[]	no
flow_log_cloudwatch_log_group_class	Specified the log class of the log group. Possible values are: STANDARD or INFREQUENT_ACCESS	string	null	no
flow_log_cloudwatch_log_group_kms_key_id	The ARN of the KMS Key to use when encrypting log data for VPC flow logs	string	null	no
flow_log_cloudwatch_log_group_name_prefix	Specifies the name prefix of CloudWatch Log Group for VPC flow logs	string	"/aws/vpc-flow-log/"	no
flow_log_cloudwatch_log_group_name_suffix	Specifies the name suffix of CloudWatch Log Group for VPC flow logs	string	""	no
flow_log_cloudwatch_log_group_retention_in_days	Specifies the number of days you want to retain log events in the specified log group for VPC flow logs	number	null	no
flow_log_cloudwatch_log_group_skip_destroy	Set to true if you do not wish the log group (and any logs it may contain) to be deleted at destroy time, and instead just remove the log group from the Terraform state	bool	false	no
flow_log_deliver_cross_account_role	(Optional) ARN of the IAM role that allows Amazon EC2 to publish flow logs across accounts.	string	null	no
flow_log_destination_arn	The ARN of the CloudWatch log group or S3 bucket where VPC Flow Logs will be pushed. If this ARN is a S3 bucket the appropriate permissions need to be set on that bucket's policy. When create_flow_log_cloudwatch_log_group is set to false this argument must be provided	string	""	no
flow_log_destination_type	Type of flow log destination. Can be s3, kinesis-data-firehose or cloud-watch-logs	string	"cloud-watch-logs"	no
flow_log_file_format	(Optional) The format for the flow log. Valid values: plain-text, parquet	string	null	no
flow_log_hive_compatible_partitions	(Optional) Indicates whether to use Hive-compatible prefixes for flow logs stored in Amazon S3	bool	false	no
flow_log_log_format	The fields to include in the flow log record, in the order in which they should appear	string	null	no
flow_log_max_aggregation_interval	The maximum interval of time during which a flow of packets is captured and aggregated into a flow log record. Valid Values: 60 seconds or 600 seconds	number	600	no
flow_log_per_hour_partition	(Optional) Indicates whether to partition the flow log per hour. This reduces the cost and response time for queries	bool	false	no
flow_log_traffic_type	The type of traffic to capture. Valid values: ACCEPT, REJECT, ALL	string	"ALL"	no
igw_tags	Additional tags for the internet gateway	map(string)	{}	no
instance_tenancy	A tenancy option for instances launched into the VPC	string	"default"	no
intra_acl_tags	Additional tags for the intra subnets network ACL	map(string)	{}	no
intra_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for intra subnets	bool	false	no
intra_inbound_acl_rules	Intra subnets inbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
intra_outbound_acl_rules	Intra subnets outbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
intra_route_table_tags	Additional tags for the intra route tables	map(string)	{}	no
intra_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
intra_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
intra_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
intra_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
intra_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
intra_subnet_ipv6_prefixes	Assigns IPv6 intra subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
intra_subnet_names	Explicit values to use in the Name tag on intra subnets. If empty, Name tags are generated	list(string)	[]	no
intra_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
intra_subnet_suffix	Suffix to append to intra subnets name	string	"intra"	no
intra_subnet_tags	Additional tags for the intra subnets	map(string)	{}	no
intra_subnets	A list of intra subnets inside the VPC	list(string)	[]	no
ipv4_ipam_pool_id	(Optional) The ID of an IPv4 IPAM pool you want to use for allocating this VPC's CIDR	string	null	no
ipv4_netmask_length	(Optional) The netmask length of the IPv4 CIDR you want to allocate to this VPC. Requires specifying a ipv4_ipam_pool_id	number	null	no
ipv6_cidr	(Optional) IPv6 CIDR block to request from an IPAM Pool. Can be set explicitly or derived from IPAM using ipv6_netmask_length	string	null	no
ipv6_cidr_block_network_border_group	By default when an IPv6 CIDR is assigned to a VPC a default ipv6_cidr_block_network_border_group will be set to the region of the VPC. This can be changed to restrict advertisement of public addresses to specific Network Border Groups such as LocalZones	string	null	no
ipv6_ipam_pool_id	(Optional) IPAM Pool ID for a IPv6 pool. Conflicts with assign_generated_ipv6_cidr_block	string	null	no
ipv6_netmask_length	(Optional) Netmask length to request from IPAM Pool. Conflicts with ipv6_cidr_block. This can be omitted if IPAM pool as a allocation_default_netmask_length set. Valid values: 56	number	null	no
manage_default_network_acl	Should be true to adopt and manage Default Network ACL	bool	true	no
manage_default_route_table	Should be true to manage default route table	bool	true	no
manage_default_security_group	Should be true to adopt and manage default security group	bool	true	no
manage_default_vpc	Should be true to adopt and manage Default VPC	bool	false	no
map_customer_owned_ip_on_launch	Specify true to indicate that network interfaces created in the subnet should be assigned a customer owned IP address. The customer_owned_ipv4_pool and outpost_arn arguments must be specified when set to true. Default is false	bool	false	no
map_public_ip_on_launch	Specify true to indicate that instances launched into the subnet should be assigned a public IP address. Default is false	bool	false	no
name	Name to be used on all the resources as identifier	string	""	no
nat_eip_tags	Additional tags for the NAT EIP	map(string)	{}	no
nat_gateway_destination_cidr_block	Used to pass a custom destination route for private NAT Gateway. If not specified, the default 0.0.0.0/0 is used as a destination route	string	"0.0.0.0/0"	no
nat_gateway_tags	Additional tags for the NAT gateways	map(string)	{}	no
one_nat_gateway_per_az	Should be true if you want only one NAT Gateway per availability zone. Requires var.azs to be set, and the number of public_subnets created to be greater than or equal to the number of availability zones specified in var.azs	bool	false	no
outpost_acl_tags	Additional tags for the outpost subnets network ACL	map(string)	{}	no
outpost_arn	ARN of Outpost you want to create a subnet in	string	null	no
outpost_az	AZ where Outpost is anchored	string	null	no
outpost_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for outpost subnets	bool	false	no
outpost_inbound_acl_rules	Outpost subnets inbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
outpost_outbound_acl_rules	Outpost subnets outbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
outpost_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
outpost_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
outpost_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
outpost_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
outpost_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
outpost_subnet_ipv6_prefixes	Assigns IPv6 outpost subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
outpost_subnet_names	Explicit values to use in the Name tag on outpost subnets. If empty, Name tags are generated	list(string)	[]	no
outpost_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
outpost_subnet_suffix	Suffix to append to outpost subnets name	string	"outpost"	no
outpost_subnet_tags	Additional tags for the outpost subnets	map(string)	{}	no
outpost_subnets	A list of outpost subnets inside the VPC	list(string)	[]	no
private_acl_tags	Additional tags for the private subnets network ACL	map(string)	{}	no
private_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for private subnets	bool	false	no
private_inbound_acl_rules	Private subnets inbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
private_outbound_acl_rules	Private subnets outbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
private_route_table_tags	Additional tags for the private route tables	map(string)	{}	no
private_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
private_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
private_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
private_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
private_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
private_subnet_ipv6_prefixes	Assigns IPv6 private subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
private_subnet_names	Explicit values to use in the Name tag on private subnets. If empty, Name tags are generated	list(string)	[]	no
private_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
private_subnet_suffix	Suffix to append to private subnets name	string	"private"	no
private_subnet_tags	Additional tags for the private subnets	map(string)	{}	no
private_subnet_tags_per_az	Additional tags for the private subnets where the primary key is the AZ	map(map(string))	{}	no
private_subnets	A list of private subnets inside the VPC	list(string)	[]	no
propagate_intra_route_tables_vgw	Should be true if you want route table propagation	bool	false	no
propagate_private_route_tables_vgw	Should be true if you want route table propagation	bool	false	no
propagate_public_route_tables_vgw	Should be true if you want route table propagation	bool	false	no
public_acl_tags	Additional tags for the public subnets network ACL	map(string)	{}	no
public_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for public subnets	bool	false	no
public_inbound_acl_rules	Public subnets inbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
public_outbound_acl_rules	Public subnets outbound network ACLs	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
public_route_table_tags	Additional tags for the public route tables	map(string)	{}	no
public_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
public_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
public_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
public_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
public_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
public_subnet_ipv6_prefixes	Assigns IPv6 public subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
public_subnet_names	Explicit values to use in the Name tag on public subnets. If empty, Name tags are generated	list(string)	[]	no
public_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
public_subnet_suffix	Suffix to append to public subnets name	string	"public"	no
public_subnet_tags	Additional tags for the public subnets	map(string)	{}	no
public_subnet_tags_per_az	Additional tags for the public subnets where the primary key is the AZ	map(map(string))	{}	no
public_subnets	A list of public subnets inside the VPC	list(string)	[]	no
putin_khuylo	Do you agree that Putin doesn't respect Ukrainian sovereignty and territorial integrity? More info: https://en.wikipedia.org/wiki/Putin_khuylo!	bool	true	no
redshift_acl_tags	Additional tags for the redshift subnets network ACL	map(string)	{}	no
redshift_dedicated_network_acl	Whether to use dedicated network ACL (not default) and custom rules for redshift subnets	bool	false	no
redshift_inbound_acl_rules	Redshift subnets inbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
redshift_outbound_acl_rules	Redshift subnets outbound network ACL rules	list(map(string))	
[
  {
    "cidr_block": "0.0.0.0/0",
    "from_port": 0,
    "protocol": "-1",
    "rule_action": "allow",
    "rule_number": 100,
    "to_port": 0
  }
]
no
redshift_route_table_tags	Additional tags for the redshift route tables	map(string)	{}	no
redshift_subnet_assign_ipv6_address_on_creation	Specify true to indicate that network interfaces created in the specified subnet should be assigned an IPv6 address. Default is false	bool	false	no
redshift_subnet_enable_dns64	Indicates whether DNS queries made to the Amazon-provided DNS Resolver in this subnet should return synthetic IPv6 addresses for IPv4-only destinations. Default: true	bool	true	no
redshift_subnet_enable_resource_name_dns_a_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS A records. Default: false	bool	false	no
redshift_subnet_enable_resource_name_dns_aaaa_record_on_launch	Indicates whether to respond to DNS queries for instance hostnames with DNS AAAA records. Default: true	bool	true	no
redshift_subnet_group_name	Name of redshift subnet group	string	null	no
redshift_subnet_group_tags	Additional tags for the redshift subnet group	map(string)	{}	no
redshift_subnet_ipv6_native	Indicates whether to create an IPv6-only subnet. Default: false	bool	false	no
redshift_subnet_ipv6_prefixes	Assigns IPv6 redshift subnet id based on the Amazon provided /56 prefix base 10 integer (0-256). Must be of equal length to the corresponding IPv4 subnet list	list(string)	[]	no
redshift_subnet_names	Explicit values to use in the Name tag on redshift subnets. If empty, Name tags are generated	list(string)	[]	no
redshift_subnet_private_dns_hostname_type_on_launch	The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: ip-name, resource-name	string	null	no
redshift_subnet_suffix	Suffix to append to redshift subnets name	string	"redshift"	no
redshift_subnet_tags	Additional tags for the redshift subnets	map(string)	{}	no
redshift_subnets	A list of redshift subnets inside the VPC	list(string)	[]	no
reuse_nat_ips	Should be true if you don't want EIPs to be created for your NAT Gateways and will instead pass them in via the 'external_nat_ip_ids' variable	bool	false	no
secondary_cidr_blocks	List of secondary CIDR blocks to associate with the VPC to extend the IP Address pool	list(string)	[]	no
single_nat_gateway	Should be true if you want to provision a single shared NAT Gateway across all of your private networks	bool	false	no
tags	A map of tags to add to all resources	map(string)	{}	no
use_ipam_pool	Determines whether IPAM pool is used for CIDR allocation	bool	false	no
vpc_block_public_access_exclusions	A map of VPC block public access exclusions	map(any)	{}	no
vpc_block_public_access_options	A map of VPC block public access options	map(string)	{}	no
vpc_flow_log_iam_policy_name	Name of the IAM policy	string	"vpc-flow-log-to-cloudwatch"	no
vpc_flow_log_iam_policy_use_name_prefix	Determines whether the name of the IAM policy (vpc_flow_log_iam_policy_name) is used as a prefix	bool	true	no
vpc_flow_log_iam_role_name	Name to use on the VPC Flow Log IAM role created	string	"vpc-flow-log-role"	no
vpc_flow_log_iam_role_use_name_prefix	Determines whether the IAM role name (vpc_flow_log_iam_role_name_name) is used as a prefix	bool	true	no
vpc_flow_log_permissions_boundary	The ARN of the Permissions Boundary for the VPC Flow Log IAM Role	string	null	no
vpc_flow_log_tags	Additional tags for the VPC Flow Logs	map(string)	{}	no
vpc_tags	Additional tags for the VPC	map(string)	{}	no
vpn_gateway_az	The Availability Zone for the VPN Gateway	string	null	no
vpn_gateway_id	ID of VPN Gateway to attach to the VPC	string	""	no
vpn_gateway_tags	Additional tags for the VPN gateway	map(string)	{}	no
Outputs
Name	Description
azs	A list of availability zones specified as argument to this module
cgw_arns	List of ARNs of Customer Gateway
cgw_ids	List of IDs of Customer Gateway
database_internet_gateway_route_id	ID of the database internet gateway route
database_ipv6_egress_route_id	ID of the database IPv6 egress route
database_nat_gateway_route_ids	List of IDs of the database nat gateway route
database_network_acl_arn	ARN of the database network ACL
database_network_acl_id	ID of the database network ACL
database_route_table_association_ids	List of IDs of the database route table association
database_route_table_ids	List of IDs of database route tables
database_subnet_arns	List of ARNs of database subnets
database_subnet_group	ID of database subnet group
database_subnet_group_name	Name of database subnet group
database_subnet_objects	A list of all database subnets, containing the full objects.
database_subnets	List of IDs of database subnets
database_subnets_cidr_blocks	List of cidr_blocks of database subnets
database_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of database subnets in an IPv6 enabled VPC
default_network_acl_id	The ID of the default network ACL
default_route_table_id	The ID of the default route table
default_security_group_id	The ID of the security group created by default on VPC creation
default_vpc_arn	The ARN of the Default VPC
default_vpc_cidr_block	The CIDR block of the Default VPC
default_vpc_default_network_acl_id	The ID of the default network ACL of the Default VPC
default_vpc_default_route_table_id	The ID of the default route table of the Default VPC
default_vpc_default_security_group_id	The ID of the security group created by default on Default VPC creation
default_vpc_enable_dns_hostnames	Whether or not the Default VPC has DNS hostname support
default_vpc_enable_dns_support	Whether or not the Default VPC has DNS support
default_vpc_id	The ID of the Default VPC
default_vpc_instance_tenancy	Tenancy of instances spin up within Default VPC
default_vpc_main_route_table_id	The ID of the main route table associated with the Default VPC
dhcp_options_id	The ID of the DHCP options
egress_only_internet_gateway_id	The ID of the egress only Internet Gateway
elasticache_network_acl_arn	ARN of the elasticache network ACL
elasticache_network_acl_id	ID of the elasticache network ACL
elasticache_route_table_association_ids	List of IDs of the elasticache route table association
elasticache_route_table_ids	List of IDs of elasticache route tables
elasticache_subnet_arns	List of ARNs of elasticache subnets
elasticache_subnet_group	ID of elasticache subnet group
elasticache_subnet_group_name	Name of elasticache subnet group
elasticache_subnet_objects	A list of all elasticache subnets, containing the full objects.
elasticache_subnets	List of IDs of elasticache subnets
elasticache_subnets_cidr_blocks	List of cidr_blocks of elasticache subnets
elasticache_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of elasticache subnets in an IPv6 enabled VPC
igw_arn	The ARN of the Internet Gateway
igw_id	The ID of the Internet Gateway
intra_network_acl_arn	ARN of the intra network ACL
intra_network_acl_id	ID of the intra network ACL
intra_route_table_association_ids	List of IDs of the intra route table association
intra_route_table_ids	List of IDs of intra route tables
intra_subnet_arns	List of ARNs of intra subnets
intra_subnet_objects	A list of all intra subnets, containing the full objects.
intra_subnets	List of IDs of intra subnets
intra_subnets_cidr_blocks	List of cidr_blocks of intra subnets
intra_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of intra subnets in an IPv6 enabled VPC
name	The name of the VPC specified as argument to this module
nat_ids	List of allocation ID of Elastic IPs created for AWS NAT Gateway
nat_public_ips	List of public Elastic IPs created for AWS NAT Gateway
natgw_ids	List of NAT Gateway IDs
natgw_interface_ids	List of Network Interface IDs assigned to NAT Gateways
outpost_network_acl_arn	ARN of the outpost network ACL
outpost_network_acl_id	ID of the outpost network ACL
outpost_subnet_arns	List of ARNs of outpost subnets
outpost_subnet_objects	A list of all outpost subnets, containing the full objects.
outpost_subnets	List of IDs of outpost subnets
outpost_subnets_cidr_blocks	List of cidr_blocks of outpost subnets
outpost_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of outpost subnets in an IPv6 enabled VPC
private_ipv6_egress_route_ids	List of IDs of the ipv6 egress route
private_nat_gateway_route_ids	List of IDs of the private nat gateway route
private_network_acl_arn	ARN of the private network ACL
private_network_acl_id	ID of the private network ACL
private_route_table_association_ids	List of IDs of the private route table association
private_route_table_ids	List of IDs of private route tables
private_subnet_arns	List of ARNs of private subnets
private_subnet_objects	A list of all private subnets, containing the full objects.
private_subnets	List of IDs of private subnets
private_subnets_cidr_blocks	List of cidr_blocks of private subnets
private_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of private subnets in an IPv6 enabled VPC
public_internet_gateway_ipv6_route_id	ID of the IPv6 internet gateway route
public_internet_gateway_route_id	ID of the internet gateway route
public_network_acl_arn	ARN of the public network ACL
public_network_acl_id	ID of the public network ACL
public_route_table_association_ids	List of IDs of the public route table association
public_route_table_ids	List of IDs of public route tables
public_subnet_arns	List of ARNs of public subnets
public_subnet_objects	A list of all public subnets, containing the full objects.
public_subnets	List of IDs of public subnets
public_subnets_cidr_blocks	List of cidr_blocks of public subnets
public_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of public subnets in an IPv6 enabled VPC
redshift_network_acl_arn	ARN of the redshift network ACL
redshift_network_acl_id	ID of the redshift network ACL
redshift_public_route_table_association_ids	List of IDs of the public redshift route table association
redshift_route_table_association_ids	List of IDs of the redshift route table association
redshift_route_table_ids	List of IDs of redshift route tables
redshift_subnet_arns	List of ARNs of redshift subnets
redshift_subnet_group	ID of redshift subnet group
redshift_subnet_objects	A list of all redshift subnets, containing the full objects.
redshift_subnets	List of IDs of redshift subnets
redshift_subnets_cidr_blocks	List of cidr_blocks of redshift subnets
redshift_subnets_ipv6_cidr_blocks	List of IPv6 cidr_blocks of redshift subnets in an IPv6 enabled VPC
this_customer_gateway	Map of Customer Gateway attributes
vgw_arn	The ARN of the VPN Gateway
vgw_id	The ID of the VPN Gateway
vpc_arn	The ARN of the VPC
vpc_block_public_access_exclusions	A map of VPC block public access exclusions
vpc_cidr_block	The CIDR block of the VPC
vpc_enable_dns_hostnames	Whether or not the VPC has DNS hostname support
vpc_enable_dns_support	Whether or not the VPC has DNS support
vpc_flow_log_cloudwatch_iam_role_arn	The ARN of the IAM role used when pushing logs to Cloudwatch log group
vpc_flow_log_deliver_cross_account_role	The ARN of the IAM role used when pushing logs cross account
vpc_flow_log_destination_arn	The ARN of the destination for VPC Flow Logs
vpc_flow_log_destination_type	The type of the destination for VPC Flow Logs
vpc_flow_log_id	The ID of the Flow Log resource
vpc_id	The ID of the VPC
vpc_instance_tenancy	Tenancy of instances spin up within VPC
vpc_ipv6_association_id	The association ID for the IPv6 CIDR block
vpc_ipv6_cidr_block	The IPv6 CIDR block
vpc_main_route_table_id	The ID of the main route table associated with this VPC
vpc_owner_id	The ID of the AWS account that owns the VPC
vpc_secondary_cidr_blocks	List of secondary CIDR blocks of the VPC
Authors
Module is maintained by Anton Babenko with help from these awesome contributors.

License
Apache 2 Licensed. See LICENSE for full details.

Additional information for users from Russia and Belarus
Russia has illegally annexed Crimea in 2014 and brought the war in Donbas followed by full-scale invasion of Ukraine in 2022.
Russia has brought sorrow and devastations to millions of Ukrainians, killed thousands of innocent people, damaged thousands of buildings including critical infrastructure, caused ecocide by blowing up a dam, bombed theater in Mariupol that had "Children" marking on the ground, raped men and boys, deported children in the occupied territoris, and forced millions of people to flee.
Putin khuylo!
