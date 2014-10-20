CloudFormation templates for a [Bamboo](https://github.com/QubitProducts/bamboo) stack proxying requests to a [Mesos](http://mesos.apache.org)+[Marathon](https://github.com/mesosphere/marathon) cluster.

Prerequisites:
* An Exhibitor-managed ZooKeeper cluster, such as provided by [thefactory/cloudformation-zookeeper](https://github.com/thefactory/cloudformation-zookeeper). Specifically, you'll need:
    - An Exhibitor endpoint for ZK node discovery
    - A ZK client security group to associate with the Mesos nodes
* A Mesos cluster running the Marathon framework, such as provided by [thefactory/cloudformation-mesos](https://github.com/thefactory/cloudformation-mesos)

## Overview

This template bootstraps a Bamboo-managed load balancing stack.

Bamboo servers are launched from public AMIs running Ubuntu 14.04 LTS and pre-loaded with Docker and Runit. If you wish to use your own image, simply modify `RegionMap` in `bamboo.json`.

Bamboo is run via a Docker image specified as a Parameter. You can use the default or provide your own.

To adjust cluster capacity, simply increment or decrement `InstanceCount` in the CloudFormation stack. Node addition/removal will be handled transparently by the auto scaling group.

Bamboo uses ZooKeeper for coordination, so the templates expect a security group (granting ZK access) and a ZK node discovery URL (exposed by Exhibitor) to be passed in.

Note that this template must be used with Amazon VPC. New AWS accounts automatically use VPC, but if you have an old account and are still using EC2-Classic, you'll need to modify this template or make the switch.

## Usage

### 1. Clone the repository
```bash
git clone git@github.com:thefactory/cloudformation-bamboo.git
```

### 2. Create an Admin security group
This is a VPC security group containing access rules for cluster administration, and should be locked down to your IP range, a bastion host, or similar. This security group will be associated with the Mesos servers.

Inbound rules are at your discretion, but you may want to include access to:
* `22 [tcp]` - SSH port
* `80 [tcp]` - Proxied HTTP
* `8000 [tcp]` - Bamboo admin HTTP

### 3. Set up ZooKeeper
You can use the instructions and template at [thefactory/cloudformation-zookeeper](https://github.com/thefactory/cloudformation-zookeeper), or you can use an existing cluster.

We'll need two things:
* `ExhibitorDiscoveryUrl` - a URL that returns a list of active ZK nodes. The expected format is that of Exhibitor's `/cluster/list` call. Example response:
```
{
    "servers": [
        "zk1.mydomain.com",
        "zk2.mydomain.com",
        "zk3.mydomain.com"
    ],
    "port": 2181
}
```
* `ZkClientSecurityGroup` - a security group that grants access to the ZooKeeper servers

If you used the aforementioned template, you can simply copy the stack's outputs.

### 4. Set up Mesos+Marathon
You can use the instructions and template at [thefactory/cloudformation-mesos](https://github.com/thefactory/cloudformation-mesos), or you can use an existing cluster.

We'll need:
* A URL associated with Marathon
* A security group associated with the Marathon servers (this will be granted access to Bamboo's callback URL)

### 5. Launch the stack
Launch the stack via the AWS console, a script, or [aws-cli](https://github.com/aws/aws-cli).

See `bamboo.json` for the full list of parameters, descriptions, and default values.

Example using `aws-cli`:
```bash
aws cloudformation create-stack \
    --template-body file://bamboo.json \
    --stack-name <stack> \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=KeyName,ParameterValue=<key> \
        ParameterKey=ExhibitorDiscoveryUrl,ParameterValue=<url> \
        ParameterKey=ZkClientSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=MarathonServerSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=VpcId,ParameterValue=<vpc_id> \
        ParameterKey=Subnets,ParameterValue='<subnet_id_1>\,<subnet_id_2>' \
        ParameterKey=AdminSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=MarathonUrl,ParameterValue=<url>
```

### 4. Watch the cluster converge
Once the stack has been provisioned, visit Bamboo's admin interface at `http://<bamboo_elb>:8000/`. You will need to do this from a location granted access by the specified `AdminSecurityGroup`.

_Note the Docker image for Bamboo may take several minutes to retrieve. This can be improved with the use of a private Docker registry._

You should be able to configure Bamboo as desired. Changes are persisted in ZK, so subsequent servers will automatically use the latest configuration.
