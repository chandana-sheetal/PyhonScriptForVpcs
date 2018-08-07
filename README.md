# PyhonScriptForVpcs
import sys
import json
import os
import argparse
import boto3
import time
import botocore
from troposphere import Join, Output, GetAZs
from troposphere import Parameter, Ref, Tags, Template, Output, Condition, Equals, And, Or, Not, If
from troposphere import Base64, FindInMap, GetAtt, Select
from troposphere.ec2 import (SecurityGroup, SecurityGroupIngress, SecurityGroupRule, Instance, NetworkInterfaceProperty, PlacementGroup, VPC,
                             Subnet, InternetGateway, VPCGatewayAttachment, RouteTable, SubnetRouteTableAssociation,
                             Route, EIP, NatGateway, BlockDeviceMapping, EBSBlockDevice, VPNGateway, CustomerGateway, VPNConnection, VpnTunnelOptionsSpecification,
                             VPNGatewayRoutePropagation)

from troposphere import autoscaling
from troposphere import cloudformation as cfn
from troposphere.autoscaling import AutoScalingGroup, Tag, LaunchConfiguration, Metadata
from troposphere.policies import CreationPolicy, ResourceSignal
import troposphere.iam as iam
from troposphere.awslambda import Function, Code
from troposphere.cloudformation import CustomResource 
from troposphere.cloudformation import AWSCustomObject
from awacs.sts import AssumeRole
from awacs.aws import Allow, Statement, Principal, Policy
from awacs import autoscaling as autoscaling
from awacs import ec2 as ec2

CIDR_PUBLIC_A = "XXXXXXXXXX"
CIDR_PUBLIC_B = "XXXXXXXXXX"
CIDR_PUBLIC_C = "XXXXXXXXXX"
CIDR_PRIVATE_A = "XXXXXXXXXX"
CIDR_PRIVATE_B = "XXXXXXXXXX"
CIDR_PRIVATE_C = "XXXXXXXXXX"
CIDR_PRIVATE_D = "XXXXXXXXXX"
CIDR_PRIVATE_E = "XXXXXXXXXX"
CIDR_PRIVATE_F = "XXXXXXXXXX"

def create_resource_tags(*args):
    if args:
        args = [Ref("AWS::StackName")] + list(args)
        return Join("-", args)
    return Ref("AWS::StackName")



def create_cfn_template(region, output_file):

    template = Template()

    template.add_description("Stack creating a VPC")

    # Internet Gateway

    internetgateway =  template.add_resource(InternetGateway(
        'InternetGateway',
        Tags=Tags(Name=create_resource_tags("InternetGW"))
    ))

   # VPC setup

    vpc = template.add_resource(VPC(
        'VPC',
        EnableDnsSupport = 'true',
        CidrBlock = '10.82.64.0/18',
        EnableDnsHostnames = 'true',
        Tags=Tags(Name=create_resource_tags("VPC"))
    ))

    internetgatewayattachment = template.add_resource(VPCGatewayAttachment(
        'InternetGatewayAttachment',
        VpcId=Ref('VPC'),
        InternetGatewayId = Ref('InternetGateway')
    ))

    public_route_table = template.add_resource(RouteTable(
        'PublicRouteTable',
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("RouteTable"))
    ))

    default_public_route = template.add_resource(Route(
        'PublicDefaultRoute',
        RouteTableId=Ref(public_route_table),
        DestinationCidrBlock='0.0.0.0/0',
        GatewayId=Ref(internetgateway),
        DependsOn=["PublicRouteTable"]
    ))

     # Create Public Subnet A
    public_subnet_a = template.add_resource(Subnet(
        'PublicSubneta',
        CidrBlock=CIDR_PUBLIC_A,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "a"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("publicSubnet-1"))
    ))

    public_route_association_a = template.add_resource(SubnetRouteTableAssociation(
        'PublicRouteAssociationA',
        SubnetId=Ref(public_subnet_a),
        RouteTableId=Ref(public_route_table),
        DependsOn=["PublicSubneta", "PublicRouteTable"]
    ))

        # Create Public Subnet B
    public_subnet_b = template.add_resource(Subnet(
        'PublicSubnetb',
        CidrBlock=CIDR_PUBLIC_B,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "b"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("publicSubnet-2"))
    ))

    public_route_association_b = template.add_resource(SubnetRouteTableAssociation(
        'PublicRouteAssociationB',
        SubnetId=Ref(public_subnet_b),
        RouteTableId=Ref(public_route_table),
        DependsOn=["PublicSubnetb", "PublicRouteTable"]
    ))

        # Create Public Subnet C
    public_subnet_c = template.add_resource(Subnet(
        'PublicSubnetc',
        CidrBlock=CIDR_PUBLIC_C,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "c"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("publicSubnet-3"))
    ))

    public_route_association_c = template.add_resource(SubnetRouteTableAssociation(
        'PublicRouteAssociationC',
        SubnetId=Ref(public_subnet_c),
        RouteTableId=Ref(public_route_table),
        DependsOn=["PublicSubnetc", "PublicRouteTable"]
    ))



    nat_eip = template.add_resource(EIP(
        'NatEip',
        Domain="vpc",
    ))

    nat_gw = template.add_resource(NatGateway(
        'NATGW',
        AllocationId=GetAtt(nat_eip, 'AllocationId'),
        SubnetId=Ref(public_subnet_a),
        DependsOn=["PublicSubneta", "PublicRouteTable", "NatEip"],
        Tags=Tags(Name=create_resource_tags("NATGateway"))
    ))

    private_route_table = template.add_resource(RouteTable(
        'PrivateRouteTable',
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("PrivateSubnetRouteTable"))
    ))

    private_route_NAT = template.add_resource(Route(
        'NatRoute',
        RouteTableId=Ref(private_route_table),
        DestinationCidrBlock='0.0.0.0/0',
        NatGatewayId=Ref(nat_gw),
        DependsOn=["PrivateRouteTable"]
    ))

        # Private subnet wo reachback-1
    private_subnet_a = template.add_resource(Subnet(
        'PrivateSubneta',
        CidrBlock=CIDR_PRIVATE_A,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "a"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-wo-Reachback-1"))
    ))

    private_route_association_a = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationA',
        SubnetId=Ref(private_subnet_a),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubneta", "PrivateRouteTable"]
    ))

        # Private subnet wo reachback-2
    private_subnet_b = template.add_resource(Subnet(
        'PrivateSubnetb',
        CidrBlock=CIDR_PRIVATE_B,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "b"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-wo-Reachback-2"))
    ))

    private_route_association_b = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationB',
        SubnetId=Ref(private_subnet_b),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubnetb", "PrivateRouteTable"]
    ))

        # Private subnet wo reachback-3
    private_subnet_c = template.add_resource(Subnet(
        'PrivateSubnetc',
        CidrBlock=CIDR_PRIVATE_C,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "c"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-wo-Reachback-3"))
    ))

    private_route_association_c = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationC',
        SubnetId=Ref(private_subnet_c),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubnetc", "PrivateRouteTable"]
    ))

        # Private subnet with reachback-1
    private_subnet_d = template.add_resource(Subnet(
        'PrivateSubnetd',
        CidrBlock=CIDR_PRIVATE_D,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "a"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-w-Reachback-1"))
    ))

    private_route_association_d = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationD',
        SubnetId=Ref(private_subnet_d),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubnetd", "PrivateRouteTable"]
    ))

        # Private subnet with reachback-2
    private_subnet_e = template.add_resource(Subnet(
        'PrivateSubnete',
        CidrBlock=CIDR_PRIVATE_E,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "b"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-w-Reachback-2"))
    ))

    private_route_association_e = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationE',
        SubnetId=Ref(private_subnet_e),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubnete", "PrivateRouteTable"]
    ))

        # Private subnet with reachback-3
    private_subnet_f = template.add_resource(Subnet(
        'PrivateSubnetf',
        CidrBlock=CIDR_PRIVATE_F,
        AvailabilityZone=Join("", [Ref("AWS::Region"), "c"]),
        VpcId=Ref("VPC"),
        Tags=Tags(Name=create_resource_tags("Private-w-Reachback-3"))
    ))

    private_route_association_f = template.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociationF',
        SubnetId=Ref(private_subnet_f),
        RouteTableId=Ref(private_route_table),
        DependsOn=["PrivateSubnetf", "PrivateRouteTable"]
    ))


    VPCId = template.add_output(Output(
        'VPCId',
        Description = 'VPCId of the newly created VPC',
        Value = Ref('VPC'),
    ))


    rendered_template = template.to_json()
    write_to_file(output_file, rendered_template)

def load_json_from_file(file_path):
    try:
        with open(file_path) as json_data:
            return json.load(json_data)
    except IOError as e:
        sys.exit(1)
    except:
        sys.exit(1)

def create_infra_stack(region, output_template, stack_name):
    try:
        client = boto3.client('cloudformation', region_name=region)
        with open(output_template, 'r') as f:
            response = client.create_stack(
               StackName=stack_name,
               TemplateBody=f.read(),
               DisableRollback=True,
               Capabilities=['CAPABILITY_IAM']
            )
        describe_stack(stack_name, region)
    except botocore.exceptions.ClientError as e:
        print("Error: ", e)
    except IOError as e:
        sys.exit(1)
    except:
        sys.exit(1)

def describe_stack(stack_name, region):
    try:
        client = boto3.client('cloudformation', region_name=region)
        describe_stack = client.describe_stacks(
            StackName=stack_name)
        stack_state = describe_stack['Stacks'][0]['StackStatus']
        while stack_state != 'CREATE_COMPLETE':
            time.sleep(10)
            describe_stack = client.describe_stacks(
            StackName=stack_name)
            stack_state = describe_stack['Stacks'][0]['StackStatus']
        output_vpc = describe_stack['Stacks'][0]['Outputs'][0]['OutputValue']
        output_subnet = describe_stack['Stacks'][0]['Outputs'][1]['OutputValue']
        d = {
             "VPC_ID": output_vpc,
             "SUBNET_ID": output_subnet
            }
        data = json.dumps(d)
        print(data)
        with open('infra_output.json', 'w') as f:
           f.write(data)
    except botocore.exceptions.ClientError as e:
        print("Error: ", e)
    except IOError as e:
        sys.exit(1)
    except:
        sys.exit(1)



def write_to_file(file_path, file_contents):
    try:
        with open(file_path, 'w') as fh:
            fh.write(file_contents)
        return None
    except IOError as e:
        print("Cannot open file for write operation. Error: %s" % (str(e)))
    except Exception as e:
        print("Unhandled exception occurred: %s" % (str(e)))
    sys.exit(1)


def cmdline_parser():
    parser = argparse.ArgumentParser()
    parser.add_argument("-r", "--region", help='Region for infrastracture setup', required=True)
    parser.add_argument("-o", "--output-json", default='./infra_template.json', help='File path to store resultant output JSON structure')
    parser.add_argument("-s", "--stack-name", help='Name of the stack to create')
    return parser.parse_args()


def main():
    args = cmdline_parser()
    print(args)
    region = args.region
    stack_name = args.stack_name
    create_cfn_template(region, args.output_json)
    create_infra_stack(region, args.output_json, stack_name)

if __name__ == "__main__":
    main()
