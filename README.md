<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/asg-elb/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# AutoScaling Instances with ELB CloudFormation Blueprint

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

![CodeBuild](https://codebuild.us-west-2.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiNmRwZVFkSHpBRlJWbTZOUVZsdUFvSzNwSjF3MzA1cnNjd1paR3VxQmtQaTFUMFhkZjBXYWkva0RsYXdSc2FpbC9sRlVrYkRXY0hTTGZWNDFlZUxOSnRBPSIsIml2UGFyYW1ldGVyU3BlYyI6InV2aUs3cHA2eUhqY3p1cUUiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)

This blueprint provisions an ELB and an AutoScaling fleet of instances. This blueprint is useful if you just need a highly available application deployed being a Load Balancer with traditional EC2 instances.

* Template makes use of the flexible AWS::AutoScaling::AutoScalingGroup MixedInstancesPolicy. This allows you to use spot, on-demand, or mixed instance types. For example, you can run 70% spot and 30% on-demand instances. Set OnDemandPercentageAboveBaseCapacity=0 to run all spot instances.
* Several [AWS::ElasticLoadBalancingV2::LoadBalancer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-loadbalancer.html) properties are configurable with [Parameters](https://lono.cloud/docs/configs/params/). Additionally, properties that require further customization are configurable with [Variables](https://lono.cloud/docs/configs/shared-variables/).  The blueprint is extremely flexible and configurable for your needs.
* You can use internal, external, application, or network Load Balancers and provision them in private or public subnets.
* You can launch the AutoScaling instances in a Custom VPC and Subnet by configuring `VpcId` and `SubnetId`.
* You can customize the UserData script and control the bootstrap process with a `@user_data_script` variable or with [configsets](https://lono.cloud/docs/configsets/). The [httpd](https://github.com/boltopspro-docs/httpd) is a good starter configset to use for testing.
* You can assign existing Security Groups to the instance or have the blueprint create a managed Security Group.
* You can optionally create a [Route53 Record](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-route53-recordset.html) and point it to the ELB dns name.

## Usage

1. Add blueprint to Gemfile
2. Configure: configs/asg-elb values
3. Deploy

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "asg-elb", git: "git@github.com:boltopspro/asg-elb.git"
```

## Configure

First you want to configure the [configs](https://lono.cloud/docs/core/configs/) files. Use [lono seed](https://lono.cloud/reference/lono-seed/) to configure starter values quickly.

    lono seed asg-elb

The generated files in `config/asg-elb` folder look something like this:

    configs/asg-elb/
    ├── params
    │   └── development.txt
    └── variables
        └── development.rb

Here's an example config with some of the values:

configs/asg-elb/params/development.txt

    # Parameter Group: AWS::AutoScaling::AutoScalingGroup
    VPCZoneIdentifier=<%= lookup_output("vpc.PrivateAppSubnets") %> # (required)
    # MaxSize=8
    # MinSize=1

    # Parameter Group: AWS::AutoScaling::AutoScalingGroup MixedInstancesPolicy InstancesDistribution
    OnDemandPercentageAboveBaseCapacity=0 # 0 means all spot instances, 100 means all on-demand instances. Default is 100
    # OnDemandBaseCapacity= # The minimum amount of the Auto Scaling group's capacity that must be fulfilled by On-Demand Instances.
    # SpotAllocationStrategy= # lowest-price | capacity-optimized
    # SpotInstancePools= # The range is 1 to 20. Higher means more spot pools.

    # Parameter Group: AWS::EC2::LaunchTemplate LaunchTemplateData
    KeyName=<%= key_pairs(/default/).first %>
    # InstanceType=t3.micro

    # Parameter Group: AWS::EC2::SecurityGroup
    VpcId=<%= lookup_output("vpc.Vpc") %> # (required)

    # Parameter Group: AWS::ElasticLoadBalancingV2::LoadBalancer
    Subnets=<%= lookup_output("vpc.PrivateAppSubnets") %> # Used for VPCZoneIdentifier of AutoScalingGroup also. # (required)
    # Scheme= # internal or internet-facing
    # SecurityGroups= # sg-111,sg-222
    # Type=application # application # application or network

    # Parameter Group: AWS::ElasticLoadBalancingV2::Listener
    # CertificateArn= # arn:aws:acm:us-west-2:112233445566:certificate/8d8919ce-a710-4050-976b-b33dbEXAMPLE
    # Port=80 # 80
    # Protocol=HTTP # HTTP # For Application ELB: HTTP | HTTPS. For Network ELB: TCP_UDP | UDP | TCP | TLS

    # Parameter Group: AWS::ElasticLoadBalancingV2::TargetGroup
    # HealthCheckPath=

    # Parameter Group: AWS::Route53::RecordSet
    # DnsName= # my-elb.example.com.
    # HostedZoneName= # example.com.


## Deploy

Use the [lono cfn deploy](http://lono.cloud/reference/lono-cfn-deploy/) command to deploy. Example:

    lono cfn deploy asg-elb --sure
    lono cfn deploy my-app --blueprint asg-elb --sure # different stack name from the blueprint name

## Configure: More Details

### httpd configset

The [httpd](https://github.com/boltopspro-docs/httpd) will configure, install, and run httpd. This is useful example for testing.  Add it with configs, example:

configs/configsets/base.rb:

```ruby
configset "httpd", resource: "LaunchTemplate"
```

### Adjust IAM Policy

You can adjust and add additional permissions to the IAM policy associated with the instance with: `@iam_managed_policy_arns` and `@iam_policies`. Example:

configs/variables/development.rb:

```ruby
@iam_policies = [
  {
    PolicyName: "extra-policy",
    PolicyDocument: {
      Version: "2012-10-17",
      Statement: [
        {
          Effect: "Allow",
          Action: [
            "events:*",
          ],
          Resource: "*"
        }
      ]
    }
  }
]
@iam_managed_policy_arns = ["arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"]
```

If you want to override and replace the IAM policies completely, use `@iam_managed_policy_arns_replace` and `@iam_policies_replace`.

### Changing VPCs

There's a CloudFormation bug where if you change the ELB Subnets, it causes a stack rollback. The workaround is to requires deleting the stack and recreating it.  Here's a command to use the same blueprint with a different stack name:

    lono cfn deploy asg-elb-2 --blueprint asg-elb

### Security Groups: ELB

You can change Security Group ingress rules for the ELB with `@elb_security_group_ingress`:

```ruby
@elb_security_group_ingress = [{
  CidrIp: "0.0.0.0/0", # another example: lookup_output("vpc.VpcCidr")
  FromPort: 80,
  IpProtocol: "tcp",
  ToPort: 80,
}]
```

This whitelist port 80 to the world.

### Security Groups: Default ELB -> EC2 Security Group Ingress

By default, [AWS::EC2::SecurityGroupIngress](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-ingress.html) rules are created to whitelist the ELB to talk to the EC2 instance on port 80. You can override the ports with the `@elb_to_ec2_ports_whitelist` variable. Examples:

configs/asg-elb/variables/development.rb:

```ruby
@elb_to_ec2_ports_whitelist = [80, 443] # Item are ports
# @elb_to_ec2_ports_whitelist = [(80..443)] # Items can be ranges
# @elb_to_ec2_ports_whitelist = (80..443) # Or single range
```

### Security Groups: EC2 Instances

You can change Security Group ingress rules for the EC2 instances `@security_group_ingress`:

```ruby
@security_group_ingress = [{
  CidrIp: "0.0.0.0/0", # another example: lookup_output("vpc.VpcCidr")
  FromPort: 22,
  IpProtocol: "tcp",
  ToPort: 22,
}]
```

This whitelist port 22 to the world.

### Route53 DNS Pretty Host Name

You can use `HostedZoneId` or `HostedZoneName` to create a pretty endpoint pointing to the ELB DNS. Example:

    DnsName=my-instance.example.com.
    HostedZoneName=example.com.
