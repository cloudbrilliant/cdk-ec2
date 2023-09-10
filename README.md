# AWS CDK - Easily create EC2 Instances

Creating and manage EC2 Instances can be complex due to many dependencies and configuration options required.

This is bare minium example which creates an EC2 instance using:

* VPC basic setup with a public subnet for SSH Access
* Security Group with allow all outbound
* Security Group Ingress Rule which allows only SSH access
* Creates a default key pair for connecting to SSH
* Instance based on t2.micro based using the latest Amazon Linux 2023 Image

Resources:

* https://docs.aws.amazon.com/cdk/v2/guide/home.html
* https://github.com/aws/aws-cdk
* cdk_ec2/cdk_ec2_stack.py (see inline)

## CDK Stack

Constructs for the stack are defined in here cdk_ec2/cdk_ec2_stack.py

Contstruct for EC2 & related aws_ec2

```python
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    CfnOutput,
    CfnTag,
)

from constructs import Construct


class CdkEc2Stack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # The code that defines your stack goes here

        # Create Basic VPC
        vpc = ec2.Vpc(
            self,
            "MyVpc",
            max_azs=2,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="public-subnet-1",
                    subnet_type=ec2.SubnetType.PUBLIC,
                    cidr_mask=24,
                )
            ],
        )

        # Create Security Group
        sec_group = ec2.SecurityGroup(
            self, "MySecurityGroup", vpc=vpc, allow_all_outbound=True
        )

        # Create Security Group Ingress Rule
        sec_group.add_ingress_rule(
            ec2.Peer.any_ipv4(), ec2.Port.tcp(22), "allow SSH access"
        )

        # Create Key Pair
        # https://docs.aws.amazon.com/cdk/api/v2/python/aws_cdk.aws_ec2/CfnKeyPair.html
        cfn_key_pair = ec2.CfnKeyPair(
            self,
            "MyCfnKeyPair",
            key_name="cdk-ec2-key-pair",
            tags=[CfnTag(key="key", value="value")],
        )

        # Create EC2 instance
        # https://docs.aws.amazon.com/cdk/api/v2/python/aws_cdk.aws_ec2/README.html
        # https://docs.aws.amazon.com/linux/al2023/ug/what-is-amazon-linux.html
        instance = ec2.Instance(
            self,
            "MyInstance",
            instance_type=ec2.InstanceType("t2.micro"),
            machine_image=ec2.MachineImage.latest_amazon_linux2023(),
            vpc=vpc,
            security_group=sec_group,
            associate_public_ip_address=True,
            key_name=cfn_key_pair.key_name,
        )

        # Output Instance ID
        CfnOutput(self, "InstanceId", value=instance.instance_id)

```

## Connect to EC2

After deploying you can access details of how to connect to your EC2 instance from within AWS Console or via a series of AWS CLI commands.

Get Instance Info

```bash
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId, InstanceType, State.Name, KeyName, PublicIpAddress, PublicDnsName]' --output table
```

List Parameters (to find CDK generated keys)

```bash
aws ssm describe-parameters
```

Download key as .pem file locally (key name string will look something like this /ec2/keypair/key-000000)

```bash
aws ssm get-parameter --name <keyname> --with-decryption --query "Parameter.Value" --output text > mykey.pem 
```

Run this command, if necessary, to ensure your key is not publicly viewable

```bash
chmod 400 mykey.pem
```

SSH into new instance (update ec2-00-00-00-00.compute-1.amazonaws.com)

```bash
ssh -i "mykey.pem" ec2-user@ec2-00-00-00-00.compute-1.amazonaws.com
```

## Welcome to your CDK Python project! (CDK Template)

This is a blank project for CDK development with Python.

The `cdk.json` file tells the CDK Toolkit how to execute your app.

This project is set up like a standard Python project.  The initialization
process also creates a virtualenv within this project, stored under the `.venv`
directory.  To create the virtualenv it assumes that there is a `python3`
(or `python` for Windows) executable in your path with access to the `venv`
package. If for any reason the automatic creation of the virtualenv fails,
you can create the virtualenv manually.

To manually create a virtualenv on MacOS and Linux:

```
$ python3 -m venv .venv
```

After the init process completes and the virtualenv is created, you can use the following
step to activate your virtualenv.

```
$ source .venv/bin/activate
```

If you are a Windows platform, you would activate the virtualenv like this:

```
% .venv\Scripts\activate.bat
```

Once the virtualenv is activated, you can install the required dependencies.

```
$ pip install -r requirements.txt
```

At this point you can now synthesize the CloudFormation template for this code.

```
$ cdk synth
```

To add additional dependencies, for example other CDK libraries, just add
them to your `setup.py` file and rerun the `pip install -r requirements.txt`
command.

## Useful commands

* `cdk ls`          list all stacks in the app
* `cdk synth`       emits the synthesized CloudFormation template
* `cdk deploy`      deploy this stack to your default AWS account/region
* `cdk diff`        compare deployed stack with current state
* `cdk docs`        open CDK documentation

Enjoy!
