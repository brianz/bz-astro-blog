---
layout: $/layouts/post.astro
title: Accessing VPC Resources from AWS Lambda
date: 2017-11-02
author: Brian Zambrano
tags: ["serverless", "aws", "vpc"]
---

I'm currently working on a book for Packt publishing titled
**Serverless Design Patterns and Best Practices**. While writing and whipping out tons
of examples is quite a bit of work, and I sometimes curse myself for agreeing to this, I'm quite
excited as I work through the chapters and as it comes together.

The first three chapters in the book cover different patterns for web applications. In two of
these three sections, the logical layer of a 3-tier web application (Presentation, Data, and Logical
layers) is, of course, Serverless. Unsurprisingly, the Lambda functions talk to a database (i.e.,
the Data Layer). Also unsurprisingly, the database is PostgreSQL via RDS.

Accessing RDS from Lambda is a very very common pattern. Of course, some Lambda functions will
need to talk to an RDS instance. It may surprise you to learn that getting Lambda functions connected to
RDS instances is not as easy as you may think.

## Create the RDS instance

Serverless via CloudFormation makes it quite easy to create an RDS instance along with the rest of
your stack. And, when I say "easy" I mean, "easy after you dig through all of the
CloudFormation docs for the resources you are working with and understand the interplay between
resources, and learn what `Fn::GetAtt` and `Ref` are and how they work in detail."

Let's work through the process of creating a PostgreSQL RDS instance via `serverless.yml` /
CloudFormation. The `resources` YAML code in `serverless.yml` is verbatim CloudFormation. To
determine how to create a particular resource, it's a matter of wading through the
CloudFormation docs. Taking a look at the
[AWS::RDS::DBInstance](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html)
docs, we'll put in the bare minimum:

```yaml
resources:
  Resources:
    RDSPostgresInstance:
      Type: AWS::RDS::DBInstance
      Properties:
        AllocatedStorage: 100
        AutoMinorVersionUpgrade: true
        AvailabilityZone: ${self:provider.region}a
        DBInstanceClass: db.t2.micro
        DBName: serverless
        DBSubnetGroupName: ?????
        Engine: postgres
        EngineVersion: 9.6.2
        MasterUsername: root
        MasterUserPassword: supersecret
        PubliclyAccessible: false
        VPCSecurityGroups:
          - ?????
```

Very quickly we find there are a couple of fields which are mysterious and call out for more
CloudFormation code. What are `DBSubnetGroup` and `VPCSecurityGroups`? These are _required_, even
though the CloudFormation docs say they are conditional. I
believe they're not _officially_ required b/c at some point in time RDS instances could be created
without a VPC. Nowadays, RDS instances must be part of some VPC, either your default VPC or
another. I'm sure there are edge cases which I don't know about, but this whole post is about
access resources in a VPC...so these fields are required.

So, what are these two things and why do we need them?

### DBSubnetGroup

As I understand it, the `DBSubnetGroup` is a resource which groups together two or more VPC subnets
and provides the RDS instance the ability to be a part of the VPC. At least two subnets are
required since `DBSubnetGroup`s require a subnet in at least two availability zones.

Subnets come in two flavors, either public or private. Public subnets in a VPC allow for
connections in/out of the public network (the internet). Private subnets do not. Most
`DBSubnetGroup`s group
together _private_ subnets, because who on earth wants to expose a database to the outside world?
Of course, if you're just playing around, you may want to do just this. But, don't expose your DB
to the world, if you care about it. You've been warned.

> Note, there is much more to it when discussing making an RDS instance publically
> accessible. The subnets which it's connected to is just one part of that discussion.

Creating one of these in CloudFormation/`serverless.yml` is pretty simple. I pick three subnets in
my VPC and set them as environment variables. Using this technique makes it trivial to switch
between projects, regions or AWS accounts. If you haven't read my
[Structuring Serverless Applications]({{< ref "structuring-serverless-applications-with-python.md" >}})
I encourage you to do so to appreciate environment variables fully.

```yaml
RDSSubnetGroup:
  Type: AWS::RDS::DBSubnetGroup
  Properties:
    DBSubnetGroupDescription: RDS Subnet Group
    SubnetIds:
      - ${env:SUBNET_ID_A}
      - ${env:SUBNET_ID_B}
      - ${env:SUBNET_ID_C}
```

### VPCSecurityGroups

Next, we turn our attention to the `VPCSecurityGroups` field. This one is pretty straightforward
if you've spent any time with EC2 Security Groups. All it is is a vanilla
[AWS::EC2::SecurityGroup](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html).
We're working with PostgreSQL, so we need to open up port `5432` or whatever port you decide to
use. This one will also need a reference to the `VpcId`...just like the subnet ids, this one is
pulled from the environment.

For Security Groups to be effective gatekeepers, one needs to attach inbound rules which state who
is allowed to connect and over what port. In this scenario, we'll permit incoming traffic to port
`5432` where the _source_ is _another_ security group. Using another Security Group as the allowed
resource is necessary since Lambda will be the
inbound actor, and there's no way we'll know what IP addresses Lambda will be coming in.
Additionally, this is simply a best practice for the dark art of Security Group management.

```yaml
RDSSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Ingress for RDS Instance
    VpcId: ${env:VPC_ID}
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: "5432"
        ToPort: "5432"
        SourceSecurityGroupId:
          Ref: ServerlessSecurityGroup
```

Down the rabbit hole we go. We now have our `RDSSecurityGroup`, but we've created a dependency on
yet _another_ resource which is the inbound security group....`ServerlessSecurityGroup`.

Before moving on, notice that in the `SourceSecurityGroupId` field above the value is
`Ref: ServerlessSecurityGroup`. `Ref` is a CloudFormation function. Each CloudFormation resource
returns something different when it's passed as an argument to `Ref`. Look at the
[AWS::EC2::SecurityGroup docs about `Ref`](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html#w2ab2c21c10d416c15)
and you'll see what it returns:

> When you specify an AWS::EC2::SecurityGroup type as an argument to the Ref function, AWS
> CloudFormation returns the security group name or the security group ID (for EC2-VPC security
> groups that are not in a default VPC).

I honestly don't know what the statement in the parenthesis is referring to. However, the `Ref:`
above works. :>)

### ServerlessSecurityGroup

This security group will not need any rules. Why is that? We will merely attach this SG to our
Lambda functions. Because our Lambda functions are associated with this SG, and because this SG
is allowed inbound access to port `5432`, our Lambdas will have network access to RDS (almost)!

```yaml
ServerlessSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: SecurityGroup for Serverless Functions
    VpcId: ${env:VPC_ID}
```

### Finalizing RDSPostgresInstance

With that, we have created the necessary resources to update the CFN `RDSPostgresInstance`
section. Here, I will only show the updates needed.

```yaml
RDSPostgresInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    DBSubnetGroupName:
      Ref: RDSSubnetGroup
    VPCSecurityGroups:
      - Fn::GetAtt: RDSSecurityGroup.GroupId
```

Again, you'll notice the use of `Ref:` and there is now a new one, `FN::GetAtt`. `GetAtt` stands
for "get attribute." Each resource in a CloudFormation template has output attributes which can be
looked up dynamically at runtime. Just as with `Ref`, each resource has various attributes which
may be looked up with this function. In the CloudFormation docs simply scroll down toward the
bottom, and you'll find what is available to you.

## Updating Lambda functions

At this point, things still wouldn't work. The reason for this is that our RDS instance is inside
of a VPC and our Lambda functions are not. Our final steps include:

- Putting our Lambda functions inside the same VPC
- Attaching `ServerlessSecurityGroup` to the Lambda functions
- Updating `iamRoleStatements` to enable the Lambda functions to create EINs, which is a
  prerequisite for Lambdas to live in a VPC

```yaml
provider:
  name: aws
  runtime: python3.6
  memorySize: 128
  region: ${env:AWS_REGION}
  timeout: 5
  environment:
    DB_USERNAME: ${env:DB_USERNAME}
    DB_PASSWORD: ${env:DB_PASSWORD}
    DB_NAME: ${env:DB_NAME}
    DB_HOST:
      Fn::GetAtt:
        - RDSPostgresInstance
        - Endpoint.Address
  vpc:
    securityGroupIds:
      - Fn::GetAtt: ServerlessSecurityGroup.GroupId
    subnetIds:
      - ${env:SUBNET_ID_A}
      - ${env:SUBNET_ID_B}
      - ${env:SUBNET_ID_C}
  iamRoleStatements:
    # Allow the lambda function permission to create EINs, which is part of the
    # AWSLambdaVPCAccessExecutionRole
    - Effect: "Allow"
      Action:
        - "ec2:CreateNetworkInterface"
        - "ec2:DescribeNetworkInterfaces"
        - "ec2:DeleteNetworkInterface"
      Resource: "*"
```

The `vpc` section accomplished the first two tasks. By putting this `vpc` section in the
`provider` section, it's applied to _all_ Lambda functions. If you don't need all Lambdas talking
to RDS, then you should attach this `vpc` to only those functions which need access.

The last part is adding three permissions in the `iamRoleStatements` section. Adding these three
permissions appends the permissions to the IAM Lamba policy which Serverless creates for us
automatically.

You may also notice how the `DB_HOST` environment variable is set using the same `GetAtt` function
above. Using this method is quite lovely since application code can get the correct db host value
from the environment without us even needing to know about it or write it down anywhere.

## Conclusion

Phew! That's a lot of work to get Lambda functions talking to RDS or any other AWS resource which
lives inside of VPCs. If you use ElastiCache or other systems in private VPC subnets, you'll need
to do the same dance. Fortunately, it's the same pattern.

What is craziest of all is that while we've allowed for network access to RDS, our Lamda functions
now _cannot_ access AWS resources which are _outside_ of this VPC including the entire internet.
For example, SNS cannot be used since it's a VPC-agnostic service. Also, if your Lambda function needed to speak
to an external API on the public internet it would not work. This can be solved as well using NAT
Gateways, which will be a topic for another time.

Happy VPC-ing!

For completeness, the final `serverless.yml` file is below, minus the Lamba functions section:

```yaml
service: rds-vpc

provider:
  name: aws
  runtime: python3.6
  memorySize: 128
  region: ${env:AWS_REGION}
  timeout: 5
  environment:
    DB_USERNAME: ${env:DB_USERNAME}
    DB_PASSWORD: ${env:DB_PASSWORD}
    DB_NAME: ${env:DB_NAME}
    DB_HOST:
      Fn::GetAtt:
        - RDSPostgresInstance
        - Endpoint.Address
  vpc:
    securityGroupIds:
      - Fn::GetAtt: ServerlessSecurityGroup.GroupId
    subnetIds:
      - ${env:SUBNET_ID_A}
      - ${env:SUBNET_ID_B}
      - ${env:SUBNET_ID_C}
  iamRoleStatements:
    # Allow the lambda function permission to create EINs, which is part of the
    # AWSLambdaVPCAccessExecutionRole
    - Effect: "Allow"
      Action:
        - "ec2:CreateNetworkInterface"
        - "ec2:DescribeNetworkInterfaces"
        - "ec2:DeleteNetworkInterface"
      Resource: "*"

resources:
  Resources:
    ServerlessSecurityGroup:
      Type: AWS::EC2::SecurityGroup
      Properties:
        GroupDescription: SecurityGroup for Serverless Functions
        VpcId: ${env:VPC_ID}
    RDSSecurityGroup:
      Type: AWS::EC2::SecurityGroup
      Properties:
        GroupDescription: Ingress for RDS Instance
        VpcId: ${env:VPC_ID}
        SecurityGroupIngress:
          - IpProtocol: tcp
            FromPort: "5432"
            ToPort: "5432"
            SourceSecurityGroupId:
              Ref: ServerlessSecurityGroup
    RDSSubnetGroup:
      Type: AWS::RDS::DBSubnetGroup
      Properties:
        DBSubnetGroupDescription: RDS Subnet Group
        SubnetIds:
          - ${env:SUBNET_ID_A}
          - ${env:SUBNET_ID_B}
          - ${env:SUBNET_ID_C}
    RDSPostgresInstance:
      Type: AWS::RDS::DBInstance
      Properties:
        AllocatedStorage: 100
        AutoMinorVersionUpgrade: true
        AvailabilityZone: ${self:provider.region}a
        DBInstanceClass: db.t2.micro
        DBName: ${env:DB_NAME}
        DBSubnetGroupName:
          Ref: RDSSubnetGroup
        Engine: postgres
        EngineVersion: 9.6.2
        MasterUsername: ${env:DB_USERNAME}
        MasterUserPassword: ${env:DB_PASSWORD}
        PubliclyAccessible: false
        VPCSecurityGroups:
          - Fn::GetAtt: RDSSecurityGroup.GroupId
```
