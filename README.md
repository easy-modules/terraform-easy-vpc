# README

```hcl
module "vpc" {
  source = "easy-modules/vpc/easy"

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
```

#### NAT Gateway Scenarios

This module supports three scenarios for creating NAT gateways. Each will be explained in further detail in the corresponding sections.

* One NAT Gateway per subnet (default behavior)
  * `enable_nat_gateway = true`
  * `single_nat_gateway = false`
  * `one_nat_gateway_per_az = false`
* Single NAT Gateway
  * `enable_nat_gateway = true`
  * `single_nat_gateway = true`
  * `one_nat_gateway_per_az = false`
* One NAT Gateway per availability zone
  * `enable_nat_gateway = true`
  * `single_nat_gateway = false`
  * `one_nat_gateway_per_az = true`

If both `single_nat_gateway` and `one_nat_gateway_per_az` are set to `true`, then `single_nat_gateway` takes precedence.

**One NAT Gateway per subnet (default)**

By default, the module will determine the number of NAT Gateways to create based on the `max()` of the private subnet lists (`database_subnets`, `elasticache_subnets`, `private_subnets`, and `redshift_subnets`). The module **does not** take into account the number of `intra_subnets`, since the latter are designed to have no Internet access via NAT Gateway. For example, if your configuration looks like the following:

```hcl
database_subnets    = ["10.0.21.0/24", "10.0.22.0/24"]
private_subnets     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24"]
intra_subnets       = ["10.0.51.0/24", "10.0.52.0/24", "10.0.53.0/24"]
```

Then `5` NAT Gateways will be created since `5` private subnet CIDR blocks were specified.

**Single NAT Gateway**

If `single_nat_gateway = true`, then all private subnets will route their Internet traffic through this single NAT gateway. The NAT gateway will be placed in the first public subnet in your `public_subnets` block.

**One NAT Gateway per availability zone**

If `one_nat_gateway_per_az = true` and `single_nat_gateway = false`, then the module will place one NAT gateway in each availability zone you specify in `var.azs`. There are some requirements around using this feature flag:

* The variable `var.azs` **must** be specified.
* The number of public subnet CIDR blocks specified in `public_subnets` **must** be greater than or equal to the number of availability zones specified in `var.azs`. This is to ensure that each NAT Gateway has a dedicated public subnet to deploy to.

#### "private" versus "intra" subnets

By default, if NAT Gateways are enabled, private subnets will be configured with routes for Internet traffic that point at the NAT Gateways configured by use of the above options.

If you need private subnets that should have no Internet routing (in the sense of [RFC1918 Category 1 subnets](https://tools.ietf.org/html/rfc1918)), `intra_subnets` should be specified. An example use case is configuration of AWS Lambda functions within a VPC, where AWS Lambda functions only need to pass traffic to internal resources or VPC endpoints for AWS services. Since AWS Lambda functions allocate Elastic Network Interfaces in proportion to the traffic received ([read more](https://docs.aws.amazon.com/lambda/latest/dg/vpc.html)), it can be useful to allocate a large private subnet for such allocations, while keeping the traffic they generate entirely internal to the VPC.

You can add additional tags with `intra_subnet_tags` as with other subnet types.

#### Public access to RDS instances

Sometimes it is handy to have public access to RDS instances (it is not recommended for production) by specifying these arguments:

```hcl
  create_database_subnet_group           = true
  create_database_subnet_route_table     = true
  create_database_internet_gateway_route = true

  enable_dns_hostnames = true
  enable_dns_support   = true
```

#### Network Access Control Lists (ACL or NACL)

This module can manage network ACL and rules. Once VPC is created, AWS creates the default network ACL, which can be controlled using this module (`manage_default_network_acl = true`).

Also, each type of subnet may have its own network ACL with custom rules per subnet. Eg, set `public_dedicated_network_acl = true` to use dedicated network ACL for the public subnets; set values of `public_inbound_acl_rules` and `public_outbound_acl_rules` to specify all the NACL rules you need to have on public subnets (see `variables.tf` for default values and structures).

By default, all subnets are associated with the default network ACL.

#### Transit Gateway (TGW) integration

It is possible to integrate this VPC module with [terraform-aws-transit-gateway module](https://github.com/terraform-aws-modules/terraform-aws-transit-gateway) which handles the creation of TGW resources and VPC attachments. See [complete example there](https://github.com/terraform-aws-modules/terraform-aws-transit-gateway/tree/master/examples/complete).

#### Requirements

| Name                                   | Version |
| -------------------------------------- | ------- |
| [terraform](./#requirement\_terraform) | >= 1.0  |
| [aws](./#requirement\_aws)             | >= 5.0  |

#### Providers

| Name                    | Version |
| ----------------------- | ------- |
| [aws](./#provider\_aws) | 5.10.0  |

#### Modules

No modules.

#### Resources

| Name                                                                                                                                                             | Type     |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [aws\_customer\_gateway.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/customer\_gateway)                                     | resource |
| [aws\_db\_subnet\_group.database](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db\_subnet\_group)                                 | resource |
| [aws\_eip.nat](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)                                                                  | resource |
| [aws\_internet\_gateway.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet\_gateway)                                     | resource |
| [aws\_nat\_gateway.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat\_gateway)                                               | resource |
| [aws\_network\_acl.database](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl)                                           | resource |
| [aws\_network\_acl.intra](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl)                                              | resource |
| [aws\_network\_acl.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl)                                            | resource |
| [aws\_network\_acl.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl)                                             | resource |
| [aws\_network\_acl\_rule.database\_inbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                      | resource |
| [aws\_network\_acl\_rule.database\_outbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                     | resource |
| [aws\_network\_acl\_rule.intra\_inbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                         | resource |
| [aws\_network\_acl\_rule.intra\_outbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                        | resource |
| [aws\_network\_acl\_rule.private\_inbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                       | resource |
| [aws\_network\_acl\_rule.private\_outbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                      | resource |
| [aws\_network\_acl\_rule.public\_inbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                        | resource |
| [aws\_network\_acl\_rule.public\_outbound](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/network\_acl\_rule)                       | resource |
| [aws\_route.database\_internet\_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)                                      | resource |
| [aws\_route.database\_nat\_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)                                           | resource |
| [aws\_route.private\_nat\_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)                                            | resource |
| [aws\_route.public\_internet\_gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)                                        | resource |
| [aws\_route\_table.database](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table)                                           | resource |
| [aws\_route\_table.intra](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table)                                              | resource |
| [aws\_route\_table.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table)                                            | resource |
| [aws\_route\_table.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table)                                             | resource |
| [aws\_route\_table\_association.database](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table\_association)                 | resource |
| [aws\_route\_table\_association.intra](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table\_association)                    | resource |
| [aws\_route\_table\_association.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table\_association)                  | resource |
| [aws\_route\_table\_association.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route\_table\_association)                   | resource |
| [aws\_subnet.database](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)                                                       | resource |
| [aws\_subnet.intra](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)                                                          | resource |
| [aws\_subnet.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)                                                        | resource |
| [aws\_subnet.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)                                                         | resource |
| [aws\_vpc.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)                                                                 | resource |
| [aws\_vpc\_ipv4\_cidr\_block\_association.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc\_ipv4\_cidr\_block\_association) | resource |
| [aws\_vpn\_gateway.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpn\_gateway)                                               | resource |
| [aws\_vpn\_gateway\_attachment.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpn\_gateway\_attachment)                       | resource |
| [aws\_vpn\_gateway\_route\_propagation.intra](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpn\_gateway\_route\_propagation)      | resource |
| [aws\_vpn\_gateway\_route\_propagation.private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpn\_gateway\_route\_propagation)    | resource |
| [aws\_vpn\_gateway\_route\_propagation.public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpn\_gateway\_route\_propagation)     | resource |

#### Inputs

| Name                                                                                               | Description                                                                                                                                                  | Type                                   | Default         | Required |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------- | --------------- | :------: |
| [amazon\_side\_asn](./#input\_amazon\_side\_asn)                                                   | The Autonomous System Number (ASN) for the Amazon side of the gateway. By default the virtual private gateway is created with the current default Amazon ASN | `string`                               | `"64512"`       |    no    |
| [azs](./#input\_azs)                                                                               | A list of availability zones names or ids in the region                                                                                                      | `list(string)`                         | `[]`            |    no    |
| [cidr](./#input\_cidr)                                                                             | The CIDR block for the VPC                                                                                                                                   | `string`                               | `"10.0.0.0/16"` |    no    |
| [create\_database\_internet\_gateway\_route](./#input\_create\_database\_internet\_gateway\_route) | Controls if an internet gateway route for public database access should be created                                                                           | `bool`                                 | `false`         |    no    |
| [create\_database\_nat\_gateway\_route](./#input\_create\_database\_nat\_gateway\_route)           | Controls if a nat gateway route should be created to give internet access to the database subnets                                                            | `bool`                                 | `false`         |    no    |
| [create\_database\_subnet\_group](./#input\_create\_database\_subnet\_group)                       | Controls if database subnet group should be created (n.b. database\_subnets must also be set)                                                                | `bool`                                 | `true`          |    no    |
| [create\_database\_subnet\_route\_table](./#input\_create\_database\_subnet\_route\_table)         | Controls if separate route table for database should be created                                                                                              | `bool`                                 | `false`         |    no    |
| [create\_igw](./#input\_create\_igw)                                                               | Controls if an Internet Gateway is created for public subnets and the related routes that connect them                                                       | `bool`                                 | `true`          |    no    |
| [create\_vpc](./#input\_create\_vpc)                                                               | A boolean flag to control if VPC should be created                                                                                                           | `bool`                                 | `true`          |    no    |
| [customer\_gateway\_tags](./#input\_customer\_gateway\_tags)                                       | Additional tags for the Customer Gateway                                                                                                                     | `map(string)`                          | `{}`            |    no    |
| [customer\_gateways](./#input\_customer\_gateways)                                                 | Maps of Customer Gateway's attributes (BGP ASN and Gateway's Internet-routable external IP address)                                                          | `map(map(any))`                        | `{}`            |    no    |
| [database\_acl\_tags](./#input\_database\_acl\_tags)                                               | Additional tags for the database subnets network ACL                                                                                                         | `map(string)`                          | `{}`            |    no    |
| [database\_dedicated\_network\_acl](./#input\_database\_dedicated\_network\_acl)                   | Whether to use dedicated network ACL (not default) and custom rules for database subnets                                                                     | `bool`                                 | `false`         |    no    |
| [database\_inbound\_acl\_rules](./#input\_database\_inbound\_acl\_rules)                           | Database subnets inbound network ACL rules                                                                                                                   | <pre><code>list(object({
</code></pre> |                 |          |

```
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
```

})) |

```
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
```

\| no | | [database\_outbound\_acl\_rules](./#input\_database\_outbound\_acl\_rules) | Database subnets outbound network ACL rules |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [database\_route\_table\_tags](./#input\_database\_route\_table\_tags) | Additional tags for the database route tables | `map(string)` | `{}` | no | | [database\_subnet\_group\_name](./#input\_database\_subnet\_group\_name) | Name of database subnet group | `string` | `null` | no | | [database\_subnet\_group\_tags](./#input\_database\_subnet\_group\_tags) | Additional tags for the database subnet group | `map(string)` | `{}` | no | | [database\_subnet\_names](./#input\_database\_subnet\_names) | Explicit values to use in the Name tag on database subnets. If empty, Name tags are generated | `list(string)` | `[]` | no | | [database\_subnet\_suffix](./#input\_database\_subnet\_suffix) | Suffix to append to database subnets name | `string` | `"db"` | no | | [database\_subnet\_tags](./#input\_database\_subnet\_tags) | Additional tags for the database subnets | `map(string)` | `{}` | no | | [database\_subnets](./#input\_database\_subnets) | A list of database subnets inside the VPC | `list(string)` | `[]` | no | | [enable\_dns\_hostnames](./#input\_enable\_dns\_hostnames) | A boolean flag to enable/disable DNS hostnames in the VPC | `bool` | `true` | no | | [enable\_dns\_support](./#input\_enable\_dns\_support) | A boolean flag to enable/disable DNS support in the VPC | `bool` | `true` | no | | [enable\_nat\_gateway](./#input\_enable\_nat\_gateway) | Should be true if you want to provision NAT Gateways for each of your private networks | `bool` | `false` | no | | [enable\_vpn\_gateway](./#input\_enable\_vpn\_gateway) | Should be true if you want to create a new VPN Gateway resource and attach it to the VPC | `bool` | `false` | no | | [external\_nat\_ip\_ids](./#input\_external\_nat\_ip\_ids) | List of EIP IDs to be assigned to the NAT Gateways (used in combination with reuse\_nat\_ips) | `list(string)` | `[]` | no | | [external\_nat\_ips](./#input\_external\_nat\_ips) | List of EIPs to be used for `nat_public_ips` output (used in combination with reuse\_nat\_ips and external\_nat\_ip\_ids) | `list(string)` | `[]` | no | | [igw\_tags](./#input\_igw\_tags) | Additional tags for the internet gateway | `map(string)` | `{}` | no | | [instance\_tenancy](./#input\_instance\_tenancy) | A tenancy option for instances launched into the VPC | `string` | `"default"` | no | | [intra\_acl\_tags](./#input\_intra\_acl\_tags) | Additional tags for the intra subnets network ACL | `map(string)` | `{}` | no | | [intra\_dedicated\_network\_acl](./#input\_intra\_dedicated\_network\_acl) | Whether to use dedicated network ACL (not default) and custom rules for intra subnets | `bool` | `false` | no | | [intra\_inbound\_acl\_rules](./#input\_intra\_inbound\_acl\_rules) | Intra subnets inbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [intra\_outbound\_acl\_rules](./#input\_intra\_outbound\_acl\_rules) | Intra subnets outbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [intra\_route\_table\_tags](./#input\_intra\_route\_table\_tags) | Additional tags for the intra route tables | `map(string)` | `{}` | no | | [intra\_subnet\_names](./#input\_intra\_subnet\_names) | Explicit values to use in the Name tag on intra subnets. If empty, Name tags are generated | `list(string)` | `[]` | no | | [intra\_subnet\_private\_dns\_hostname\_type\_on\_launch](./#input\_intra\_subnet\_private\_dns\_hostname\_type\_on\_launch) | The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: `ip-name`, `resource-name` | `string` | `null` | no | | [intra\_subnet\_suffix](./#input\_intra\_subnet\_suffix) | Suffix to append to intra subnets name | `string` | `"intra"` | no | | [intra\_subnet\_tags](./#input\_intra\_subnet\_tags) | Additional tags for the intra subnets | `map(string)` | `{}` | no | | [intra\_subnets](./#input\_intra\_subnets) | A list of intra subnets inside the VPC | `list(string)` | `[]` | no | | [map\_public\_ip\_on\_launch](./#input\_map\_public\_ip\_on\_launch) | Specify true to indicate that instances launched into the subnet should be assigned a public IP address. Default is `false` | `bool` | `false` | no | | [name](./#input\_name) | Name to be used on all the resources as identifier | `string` | `""` | no | | [nat\_eip\_tags](./#input\_nat\_eip\_tags) | Additional tags for the NAT EIP | `map(string)` | `{}` | no | | [nat\_gateway\_destination\_cidr\_block](./#input\_nat\_gateway\_destination\_cidr\_block) | Used to pass a custom destination route for private NAT Gateway. If not specified, the default 0.0.0.0/0 is used as a destination route | `string` | `"0.0.0.0/0"` | no | | [one\_nat\_gateway\_per\_az](./#input\_one\_nat\_gateway\_per\_az) | Should be true if you want only one NAT Gateway per availability zone. Requires `var.azs` to be set, and the number of `public_subnets` created to be greater than or equal to the number of availability zones specified in `var.azs` | `bool` | `false` | no | | [private\_acl\_tags](./#input\_private\_acl\_tags) | Additional tags for the private subnets network ACL | `map(string)` | `{}` | no | | [private\_dedicated\_network\_acl](./#input\_private\_dedicated\_network\_acl) | Whether to use dedicated network ACL (not default) and custom rules for private subnets | `bool` | `false` | no | | [private\_inbound\_acl\_rules](./#input\_private\_inbound\_acl\_rules) | Private subnets inbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [private\_outbound\_acl\_rules](./#input\_private\_outbound\_acl\_rules) | Private subnets outbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [private\_route\_table\_tags](./#input\_private\_route\_table\_tags) | Additional tags for the private route tables | `map(string)` | `{}` | no | | [private\_subnet\_names](./#input\_private\_subnet\_names) | Explicit values to use in the Name tag on private subnets. If empty, Name tags are generated | `list(string)` | `[]` | no | | [private\_subnet\_suffix](./#input\_private\_subnet\_suffix) | Suffix to append to private subnets name | `string` | `"private"` | no | | [private\_subnet\_tags](./#input\_private\_subnet\_tags) | Additional tags for the private subnets | `map(string)` | `{}` | no | | [private\_subnet\_tags\_per\_az](./#input\_private\_subnet\_tags\_per\_az) | Additional tags for the private subnets where the primary key is the AZ | `map(map(string))` | `{}` | no | | [private\_subnets](./#input\_private\_subnets) | A list of private subnets inside the VPC | `list(string)` | `[]` | no | | [propagate\_intra\_route\_tables\_vgw](./#input\_propagate\_intra\_route\_tables\_vgw) | Should be true if you want route table propagation | `bool` | `false` | no | | [propagate\_private\_route\_tables\_vgw](./#input\_propagate\_private\_route\_tables\_vgw) | Should be true if you want route table propagation | `bool` | `false` | no | | [propagate\_public\_route\_tables\_vgw](./#input\_propagate\_public\_route\_tables\_vgw) | Should be true if you want route table propagation | `bool` | `false` | no | | [public\_dedicated\_network\_acl](./#input\_public\_dedicated\_network\_acl) | Whether to use dedicated network ACL (not default) and custom rules for public subnets | `bool` | `false` | no | | [public\_inbound\_acl\_rules](./#input\_public\_inbound\_acl\_rules) | Public subnets inbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [public\_network\_acl\_tags](./#input\_public\_network\_acl\_tags) | Additional tags for the public network ACL | `map(string)` | `{}` | no | | [public\_outbound\_acl\_rules](./#input\_public\_outbound\_acl\_rules) | Public subnets outbound network ACLs |

```
list(object({
rule_number = number
rule_action = string
from_port   = number
to_port     = number
protocol    = string
cidr_block  = string
}))
```

|

```
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
```

\| no | | [public\_route\_table\_tags](./#input\_public\_route\_table\_tags) | Additional tags for the public route tables | `map(string)` | `{}` | no | | [public\_subnet\_names](./#input\_public\_subnet\_names) | Explicit values to use in the Name tag on public subnets. If empty, Name tags are generated | `list(string)` | `[]` | no | | [public\_subnet\_private\_dns\_hostname\_type\_on\_launch](./#input\_public\_subnet\_private\_dns\_hostname\_type\_on\_launch) | The type of hostnames to assign to instances in the subnet at launch. For IPv6-only subnets, an instance DNS name must be based on the instance ID. For dual-stack and IPv4-only subnets, you can specify whether DNS names use the instance IPv4 address or the instance ID. Valid values: `ip-name`, `resource-name` | `string` | `null` | no | | [public\_subnet\_suffix](./#input\_public\_subnet\_suffix) | Suffix to append to public subnets name | `string` | `"public"` | no | | [public\_subnet\_tags](./#input\_public\_subnet\_tags) | Additional tags for the public subnets | `map(string)` | `{}` | no | | [public\_subnet\_tags\_per\_az](./#input\_public\_subnet\_tags\_per\_az) | Additional tags for the public subnets where the primary key is the AZ | `map(map(string))` | `{}` | no | | [public\_subnets](./#input\_public\_subnets) | A list of public subnets inside the VPC | `list(string)` | `[]` | no | | [reuse\_nat\_ips](./#input\_reuse\_nat\_ips) | Should be true if you don't want EIPs to be created for your NAT Gateways and will instead pass them in via the 'external\_nat\_ip\_ids' variable | `bool` | `false` | no | | [secondary\_cidr\_blocks](./#input\_secondary\_cidr\_blocks) | List of secondary CIDR blocks to associate with the VPC to extend the IP Address pool | `list(string)` | `[]` | no | | [single\_nat\_gateway](./#input\_single\_nat\_gateway) | Should be true if you want to provision a single shared NAT Gateway across all of your private networks | `bool` | `false` | no | | [tags](./#input\_tags) | A map of tags to add to all resources | `map(string)` | `{}` | no | | [vpc\_tags](./#input\_vpc\_tags) | A map of tags to add to all resources | `map(string)` | `{}` | no | | [vpn\_gateway\_az](./#input\_vpn\_gateway\_az) | The Availability Zone for the VPN Gateway | `string` | `null` | no | | [vpn\_gateway\_id](./#input\_vpn\_gateway\_id) | ID of VPN Gateway to attach to the VPC | `string` | `""` | no | | [vpn\_gateway\_tags](./#input\_vpn\_gateway\_tags) | Additional tags for the VPN gateway | `map(string)` | `{}` | no |

#### Outputs

| Name                                                                                          | Description                                                      |
| --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| [database\_internet\_gateway\_route\_id](./#output\_database\_internet\_gateway\_route\_id)   | ID of the database internet gateway route                        |
| [database\_nat\_gateway\_route\_ids](./#output\_database\_nat\_gateway\_route\_ids)           | List of IDs of the database nat gateway route                    |
| [database\_network\_acl\_arn](./#output\_database\_network\_acl\_arn)                         | ARN of the database network ACL                                  |
| [database\_network\_acl\_id](./#output\_database\_network\_acl\_id)                           | ID of the database network ACL                                   |
| [database\_route\_table\_ids](./#output\_database\_route\_table\_ids)                         | List of IDs of database route tables                             |
| [database\_subnet\_arns](./#output\_database\_subnet\_arns)                                   | List of ARNs of database subnets                                 |
| [database\_subnet\_group](./#output\_database\_subnet\_group)                                 | ID of database subnet group                                      |
| [database\_subnet\_group\_name](./#output\_database\_subnet\_group\_name)                     | Name of database subnet group                                    |
| [database\_subnets](./#output\_database\_subnets)                                             | List of IDs of database subnets                                  |
| [database\_subnets\_cidr\_blocks](./#output\_database\_subnets\_cidr\_blocks)                 | List of cidr\_blocks of database subnets                         |
| [igw\_arn](./#output\_igw\_arn)                                                               | The ARN of the Internet Gateway                                  |
| [igw\_id](./#output\_igw\_id)                                                                 | The ID of the Internet Gateway                                   |
| [intra\_network\_acl\_arn](./#output\_intra\_network\_acl\_arn)                               | ARN of the intra network ACL                                     |
| [intra\_network\_acl\_id](./#output\_intra\_network\_acl\_id)                                 | ID of the intra network ACL                                      |
| [intra\_route\_table\_association\_ids](./#output\_intra\_route\_table\_association\_ids)     | List of IDs of the intra route table association                 |
| [intra\_route\_table\_ids](./#output\_intra\_route\_table\_ids)                               | List of IDs of intra route tables                                |
| [intra\_subnet\_arns](./#output\_intra\_subnet\_arns)                                         | List of ARNs of intra subnets                                    |
| [intra\_subnets](./#output\_intra\_subnets)                                                   | List of IDs of intra subnets                                     |
| [intra\_subnets\_cidr\_blocks](./#output\_intra\_subnets\_cidr\_blocks)                       | List of cidr\_blocks of intra subnets                            |
| [nat\_ids](./#output\_nat\_ids)                                                               | List of allocation ID of Elastic IPs created for AWS NAT Gateway |
| [nat\_public\_ips](./#output\_nat\_public\_ips)                                               | List of public Elastic IPs created for AWS NAT Gateway           |
| [natgw\_ids](./#output\_natgw\_ids)                                                           | List of NAT Gateway IDs                                          |
| [private\_nat\_gateway\_route\_ids](./#output\_private\_nat\_gateway\_route\_ids)             | List of IDs of the private nat gateway route                     |
| [private\_network\_acl\_arn](./#output\_private\_network\_acl\_arn)                           | ARN of the private network ACL                                   |
| [private\_network\_acl\_id](./#output\_private\_network\_acl\_id)                             | ID of the private network ACL                                    |
| [private\_route\_table\_association\_ids](./#output\_private\_route\_table\_association\_ids) | List of IDs of the private route table association               |
| [private\_route\_table\_ids](./#output\_private\_route\_table\_ids)                           | List of IDs of private route tables                              |
| [private\_subnet\_arns](./#output\_private\_subnet\_arns)                                     | List of ARNs of the private subnets                              |
| [private\_subnets](./#output\_private\_subnets)                                               | List of IDs of private subnets                                   |
| [private\_subnets\_cidr\_blocks](./#output\_private\_subnets\_cidr\_blocks)                   | List of cidr\_blocks of private subnets                          |
| [public\_internet\_gateway\_route\_id](./#output\_public\_internet\_gateway\_route\_id)       | ID of the internet gateway route                                 |
| [public\_network\_acl\_arn](./#output\_public\_network\_acl\_arn)                             | ARN of the public network ACL                                    |
| [public\_network\_acl\_id](./#output\_public\_network\_acl\_id)                               | ID of the public network ACL                                     |
| [public\_route\_table\_association\_ids](./#output\_public\_route\_table\_association\_ids)   | List of IDs of the public route table association                |
| [public\_route\_table\_ids](./#output\_public\_route\_table\_ids)                             | List of IDs of public route tables                               |
| [public\_subnet\_arns](./#output\_public\_subnet\_arns)                                       | List of ARNs of public subnets                                   |
| [public\_subnets](./#output\_public\_subnets)                                                 | List of IDs of public subnets                                    |
| [public\_subnets\_cidr\_blocks](./#output\_public\_subnets\_cidr\_blocks)                     | List of cidr\_blocks of public subnets                           |
| [vpc\_arn](./#output\_vpc\_arn)                                                               | The ARN of the VPC                                               |
| [vpc\_cidr\_block](./#output\_vpc\_cidr\_block)                                               | The CIDR block of the VPC                                        |
| [vpc\_enable\_dns\_hostnames](./#output\_vpc\_enable\_dns\_hostnames)                         | Whether or not the VPC has DNS hostname support                  |
| [vpc\_enable\_dns\_support](./#output\_vpc\_enable\_dns\_support)                             | Whether or not the VPC has DNS support                           |
| [vpc\_id](./#output\_vpc\_id)                                                                 | The ID of the VPC                                                |
| [vpc\_main\_route\_table\_id](./#output\_vpc\_main\_route\_table\_id)                         | The ID of the main route table associated with this VPC          |
| [vpc\_owner\_id](./#output\_vpc\_owner\_id)                                                   | The ID of the AWS account that owns the VPC                      |
| [vpc\_secondary\_cidr\_blocks](./#output\_vpc\_secondary\_cidr\_blocks)                       | List of secondary CIDR blocks of the VPC                         |
|                                                                                               |                                                                  |
