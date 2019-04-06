
Amazon ECS+RDS
==============

### Overview[¶](#overview "Permanent link")

Every component (apart from the database) in this stack is packaged as a docker container. Because of this, it can be deployed to any infrastructure capable of running containers. In this section we'll explain how to setup your own infrastructure where [Amazon EC2 Container Service (ECS)](https://aws.amazon.com/ecs/) will run your containers and setup a PostgreSQL database in [Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) that will hold your data. This infrastructure will be capable of supporting multiple applications and scale in time.

### Familiarize yourself with ECS concepts and AWS[¶](#familiarize-yourself-with-ecs-concepts-and-aws "Permanent link")

*   [What is Amazon EC2 Container Service?](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
*   [Amazon EC2 Container Service](https://aws.amazon.com/ecs/)
*   [ECS - Getting Starterd](https://aws.amazon.com/ecs/getting-started/)
*   [Installing the AWS Command Line Interface](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

If you feel overwelmed by all the AWS services and their names, check out [Amazon Web Services in Plain English](https://www.expeditedssl.com/aws-in-plain-english)

### Install AWS CLI[¶](#install-aws-cli "Permanent link")

We'll be interacting with AWS mostly using it's command line interface. You will need to [install](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and [configure](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) it.

### ECS Cluster[¶](#ecs-cluster "Permanent link")

Create your cluster using the wizard (start with one ec2 instance) In this example the name used is `mycluster` [Creating a Cluster - Guide](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/create_cluster.html)

Note

On the guide page, in the first paragraph, you will be directed to [Setting Up with Amazon ECS](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/get-set-up-for-amazon-ecs.html). **DO NOT** follow those step, the docs are a bit outdated.

You probably already have a AWS account (#1) and we already installed the CLI (#7). The only step that you could follow is [\# Create a Key Pair](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/get-set-up-for-amazon-ecs.html#create-a-key-pair) if you want to ssh into instances from your cluster.

Also note, for the Create Cluster form to appear the you must ignore the first-run wizard (it appears with two checkboxes), if not it will be presented with unneeded forms for the completion of the tutorial.

This is how the form should look

![cluster-template](../../images/ecs/cluster-template.png) ![create-cluster](../../images/ecs/create-cluster.png)

From this step forward, we'll use the command line.

Save the cluster name in a env var.

export CLUSTER\_NAME\=mycluster

Get the cluster's cloudformation stack name:

aws cloudformation list-stacks --output table --query 'StackSummaries\[\*\].\[StackName,TemplateDescription\]'

Result should look something like this

\--------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                      ListStacks                                                                      |
+-------------------------------+----------------------------------------------------------------------------------------------------------------------+
|  EC2ContainerService-mycluster|  AWS CloudFormation template to create a new VPC or use an existing VPC for ECS deployment in Create Cluster Wizard  |
+-------------------------------+----------------------------------------------------------------------------------------------------------------------+

Save the stack name to env

export STACK\_NAME\=EC2ContainerService-mycluster

Extract stack configuration info into env vars

while read k v ; do export Cluster\_Resource\_$k\=$v; done < <( \\
    aws cloudformation describe-stack-resources \\
        --stack-name $STACK\_NAME \\
        --output text \\
        --query 'StackResources\[\*\].\[LogicalResourceId, PhysicalResourceId\]'\\
)

while read k v ; do export Cluster\_Param\_$k\=$v; done < <( \\
    aws cloudformation describe-stacks \\
        --stack-name $STACK\_NAME \\
        --output text \\
        --query 'Stacks\[\*\].Parameters\[\*\]\[ParameterKey,ParameterValue\]'\\
)

check if it worked with

env | grep Cluster

\#### sample output below
Cluster\_Param\_SubnetCidr2\=10.0.1.0/24
Cluster\_Param\_EbsVolumeSize\=22
Cluster\_Param\_SubnetCidr3\=
Cluster\_Param\_EcsEndpoint\=
Cluster\_Resource\_EcsInstanceAsg\=EC2ContainerService-mycluster-EcsInstanceAsg-XXXXXXXX
Cluster\_Resource\_PubSubnet2RouteTableAssociation\=rtbassoc-0000000
Cluster\_Param\_AsgMaxSize\=2
Cluster\_Param\_SubnetCidr1\=10.0.0.0/24
Cluster\_Param\_SecurityIngressToPort\=80
Cluster\_Param\_EcsClusterName\=mycluster
Cluster\_Param\_EbsVolumeType\=gp2
Cluster\_Resource\_RouteViaIgw\=rtb-aaaaaaa
Cluster\_Resource\_Vpc\=vpc-00000000
Cluster\_Param\_VpcCidr\=10.0.0.0/16
Cluster\_Param\_VpcId\=
Cluster\_Resource\_PubSubnet1RouteTableAssociation\=rtbassoc-0000000
Cluster\_Resource\_AttachGateway\=EC2Co-Attac-AAAAAAAAAA
Cluster\_Resource\_PubSubnetAz1\=subnet-aaaaaaa
Cluster\_Param\_KeyName\=mycluster-cluster
Cluster\_Resource\_PubSubnetAz2\=subnet-bbbbbbb
Cluster\_Resource\_InternetGateway\=igw-aaaaaaa
Cluster\_Param\_IamRoleInstanceProfile\=ecsInstanceRole
Cluster\_Param\_SecurityIngressFromPort\=80
Cluster\_Param\_DeviceName\=/dev/xvdcz
Cluster\_Param\_VpcAvailabilityZones\=us-east-1a,us-east-1d,us-east-1e,us-east-1c,us-east-1b
Cluster\_Param\_SecurityGroupId\=
Cluster\_Resource\_PublicRouteViaIgw\=EC2Co-Publi-AAAAAAAAAAA
Cluster\_Param\_SubnetIds\=
Cluster\_Resource\_EcsSecurityGroup\=sg-0000000
Cluster\_Resource\_EcsInstanceLc\=EC2ContainerService-mycluster-EcsInstanceLc-AAAAAAAAAA
Cluster\_Param\_EcsAmiId\=ami-04351e12
Cluster\_Param\_EcsInstanceType\=t2.medium
Cluster\_Param\_SecurityIngressCidrIp\=0.0.0.0/0

Save the cluster region in a env var

export AWS\_REGION\=\`echo $Cluster\_Param\_VpcAvailabilityZones | cut -d',' -f1 | head --bytes -2\`
echo $AWS\_REGION

### SSL Certificate[¶](#ssl-certificate "Permanent link")

If you want to use HTTPS, you will need a SSL certificate and you will need to complete this step before the next one (creating the loadbalancer). You can create (or upload) a certificate in [AWS Certificate Manager](http://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html). You must request (or configure) your certificate in the same region as your cluster, you can not use them across regions.

\# List the certificates
aws acm list-certificates

\# Save the ARN
export CERTIFICATE\_ARN\="arn:aws:acm:us-east-1:CHANGE-WITH-YOURS:certificate/CHANGE-WITH-YOURS"

Result should look like this

\------------------------------------------------------------------------------------------------------------
|                                             ListCertificates                                             |
+----------------------------------------------------------------------------------------------------------+
||                                         CertificateSummaryList                                         ||
|+---------------------------------------------------------------------------------------+----------------+|
||                                    CertificateArn                                     |  DomainName    ||
|+---------------------------------------------------------------------------------------+----------------+|
||  arn:aws:acm:us-east-1:000000000000:certificate/00000000-0000-0000-0000-000000000000  |  mydomain.com  ||
|+---------------------------------------------------------------------------------------+----------------+|

### Loadbalancer[¶](#loadbalancer "Permanent link")

The loadbalancer will route traffic to our containers (just like the cluster, it can be used for multiple applications). If you do not need HTTPS, and you skipped the previous step, make sure you set `CERTIFICATE_ARN` to empty string

export CERTIFICATE\_ARN\=""

Create the loadbalancer

curl -SLO https://docs.subzero.cloud/cloudformation/loadbalancer.yml

aws cloudformation create-stack \\
    --stack-name $CLUSTER\_NAME\-loadbalancer \\
    --template-body file://loadbalancer.yml \\
    --capabilities CAPABILITY\_IAM \\
    --parameters \\
    ParameterKey\=ClusterName,ParameterValue\=$CLUSTER\_NAME \\
    ParameterKey\=CertificateArn,ParameterValue\=$CERTIFICATE\_ARN \\
    ParameterKey\=Vpc,ParameterValue\=$Cluster\_Resource\_Vpc \\
    ParameterKey\=EcsSecurityGroup,ParameterValue\=$Cluster\_Resource\_EcsSecurityGroup \\
    ParameterKey\=PubSubnetAz1,ParameterValue\=$Cluster\_Resource\_PubSubnetAz1 \\
    ParameterKey\=PubSubnetAz2,ParameterValue\=$Cluster\_Resource\_PubSubnetAz2

### Image Repository[¶](#image-repository "Permanent link")

We'll store our OpenResty image (the only one that changes) in [Amazon EC2 Container Registry](https://aws.amazon.com/ecr/). You can use any docker image repository you like.

\# read project env vars
source .env

\# create the repository
aws ecr create-repository --repository-name $COMPOSE\_PROJECT\_NAME/openresty

\# extract the uri in a separate env var
export OPENRESTY\_REPO\_URI\=\`aws ecr describe-repositories --repository-name $COMPOSE\_PROJECT\_NAME/openresty --output text --query 'repositories\[0\].repositoryUri'\`

### Database (PostgreSQL in RDS)[¶](#database-postgresql-in-rds "Permanent link")

\# read project env vars
source .env

\# we will place the database in the same security group as our ECS cluster
export PRODUCTION\_DB\_SECURITY\_GROUP\=$Cluster\_Resource\_EcsSecurityGroup

\# set the subnet to the same VPS as the cluster
aws rds create-db-subnet-group \\
    --db-subnet-group-name $COMPOSE\_PROJECT\_NAME\-db-subnet \\
    --db-subnet-group-description $COMPOSE\_PROJECT\_NAME\-db-subnet \\
    --subnet-ids $Cluster\_Resource\_PubSubnetAz1 $Cluster\_Resource\_PubSubnetAz2

\# allow ECS nodes to connect to this db's in the same security group as the cluster
aws ec2 authorize-security-group-ingress \\
              --region $AWS\_REGION \\
              --group-id $PRODUCTION\_DB\_SECURITY\_GROUP \\
              --protocol tcp \\
              --port 5432 \\
              --source-group $PRODUCTION\_DB\_SECURITY\_GROUP

\# get your current IP
\# option A
export MY\_IP\=\`curl http://checkip.amazonaws.com/\`
\# option B
export MY\_IP\=\`dig +short myip.opendns.com @resolver1.opendns.com\`

\# allow yourself to connect directly to the database
aws ec2 authorize-security-group-ingress \\
              --region $AWS\_REGION \\
              --group-id $PRODUCTION\_DB\_SECURITY\_GROUP \\
              --protocol tcp \\
              --port 5432 \\
              --cidr $MY\_IP/32

\# create the database
aws rds create-db-instance \\
    --db-instance-identifier $COMPOSE\_PROJECT\_NAME\-db \\
    --db-name $DB\_NAME \\
    --vpc-security-group-ids $PRODUCTION\_DB\_SECURITY\_GROUP \\
    --allocated-storage 20 \\
    --db-instance-class db.t2.micro \\
    --engine postgres \\
    --publicly-accessible \\
    --multi-az \\
    --db-subnet-group-name $COMPOSE\_PROJECT\_NAME\-db-subnet \\
    --master-username $SUPER\_USER \\
    --master-user-password SET-YOUR-RDS-MASTER-PASSWORD-HERE

\# check the AWS management panel and wait until the database is "ready"

\# export production db host
export PRODUCTION\_DB\_HOST\=\`aws rds describe-db-instances --db-instance-identifier $COMPOSE\_PROJECT\_NAME\-db --output text --query 'DBInstances\[0\].Endpoint.Address'\`

\# check you can connect
psql -h $PRODUCTION\_DB\_HOST -U $SUPER\_USER $DB\_NAME

\# create the authenticator role used by PostgREST to connect
psql \\
    -h $PRODUCTION\_DB\_HOST \\
    -U $SUPER\_USER \\
    -c "create role $DB\_USER with login password 'SET-YOUR-AUTHENTICATOR-PASSWORD';" $DB\_NAME

### Application Stack[¶](#application-stack "Permanent link")

Create the application cloudformation stack.

Right now we will use `DesiredCount=0` since our application is not deployed yet (db is empty and the OpenResty images are not uploaded)

curl -SLO https://docs.subzero.cloud/cloudformation/application.yml

aws cloudformation create-stack \\
--stack-name $COMPOSE\_PROJECT\_NAME \\
--template-body file://application.yml \\
--capabilities CAPABILITY\_IAM \\
--parameters \\
ParameterKey\=ClusterName,ParameterValue\=$CLUSTER\_NAME \\
ParameterKey\=DesiredCount,ParameterValue\=0 \\
ParameterKey\=Version,ParameterValue\=v0.0.0 \\
ParameterKey\=OpenRestyImage,ParameterValue\=$OPENRESTY\_REPO\_URI \\
ParameterKey\=ListenerHostNamePattern,ParameterValue\=mydomain.com \\
ParameterKey\=HasHttpsListener,ParameterValue\=Yes \\
ParameterKey\=DbHost,ParameterValue\=$PRODUCTION\_DB\_HOST \\
ParameterKey\=DbPassword,ParameterValue\=SET-YOUR-AUTHENTICATOR-PASSWORD \\
ParameterKey\=JwtSecret,ParameterValue\=SET-YOUR-JWT-SECRET

### Production configuration file[¶](#production-configuration-file "Permanent link")

In the root folder of your application create the `subzero-app.json` configuration file like this

SUBZERO\_APP\_CONF\=$(cat <<EOF
{
 "name": "subzerostarterkit",
 "domain": "mydomain.com",
 "openresty\_repo": "${OPENRESTY\_REPO\_URI}",
 "db\_location": "external",
 "db\_admin": "${SUPER\_USER}",
 "db\_host": "${PRODUCTION\_DB\_HOST}",
 "db\_port": 5432,
 "db\_name": "${DB\_NAME}",
 "version": "v0.0.1"
}
EOF
)

echo "${SUBZERO\_APP\_CONF}" > subzero-app.json

### Deploy[¶](#deploy "Permanent link")

Make sure you have the database migrations folder ready for deploying it to the production RDS PostgreSQL instance (see [Managing Migrations](../managing-migrations/) for details).

Now for the last part of the deployment do:

aws ecr get-login --region $AWS\_REGION --no-include-email | sh

subzero cloud app-deploy --dba $SUPER\_USER --password SET-YOUR-RDS-MASTER-PASSWORD-HERE

Sending build context to Docker daemon  40.96kB                                                                                                                
Step 1/5 : FROM openresty/openresty:jessie                                                                                                                     
 ---> 2651c15a0658                                                                                                                                             
Step 2/5 : COPY entrypoint.sh /entrypoint.sh                                                                                                                   
 ---> dffa292f13af
Step 3/5 : COPY nginx /usr/local/openresty/nginx
 ---> d6f39fd98225
Step 4/5 : COPY lualib /usr/local/openresty/lualib
 ---> 6bf94d8d6a97
Step 5/5 : ENTRYPOINT \["/entrypoint.sh"\]
 ---> Running in 7d505af48a50
 ---> 036d363f58e9
Removing intermediate container 7d505af48a50
Successfully built 036d363f58e9
Successfully tagged openresty:latest

The push refers to a repository \[registry.subzero.cloud/appid-b52dcecc-f700-4309-9a61-38af41508030/openresty\]
9c774229090e: Preparing
b9ba72e6327c: Preparing
556c7a517ae4: Preparing
1b9e22042c7f: Preparing
5d6cbe0dbcf9: Preparing
556c7a517ae4: Pushed
9c774229090e: Pushed
b9ba72e6327c: Pushed
5d6cbe0dbcf9: Pushed
1b9e22042c7f: Pushed
v0.0.1: digest: sha256:b851e387d86c7458473102047cd3f4d6fcf5d0894b080fd2db169b93bbeb07ef size: 1365

Adding registry tables to db:pg://superuser:@ip-54-175-00-00.db.subzero.cloud:33544/app
Deploying changes to db:pg://superuser:@ip-54-175-00-00.db.subzero.cloud:33544/app
  + 0000000001\-initial .. ok

psql:deploy/0000000001-initial.sql:22: NOTICE:  role "authenticator" is already a member of role "anonymous"

Now bring up the docker containers by setting `DesiredCount=1`

aws cloudformation update-stack \\
--stack-name $COMPOSE\_PROJECT\_NAME \\
--template-body file://application.yml \\
--capabilities CAPABILITY\_IAM \\
--parameters \\
ParameterKey\=ClusterName,UsePreviousValue\=true \\
ParameterKey\=DesiredCount,ParameterValue\=1 \\
ParameterKey\=Version,ParameterValue\=v0.0.1 \\
ParameterKey\=OpenRestyImage,UsePreviousValue\=true \\
ParameterKey\=ListenerHostNamePattern,UsePreviousValue\=true \\
ParameterKey\=HasHttpsListener,UsePreviousValue\=true \\
ParameterKey\=DbHost,UsePreviousValue\=true \\
ParameterKey\=DbPassword,UsePreviousValue\=true \\
ParameterKey\=JwtSecret,UsePreviousValue\=true
