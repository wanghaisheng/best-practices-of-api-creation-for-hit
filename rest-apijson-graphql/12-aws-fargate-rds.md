
============================================================

### Overview[¶](#overview "Permanent link")

Every component (apart from the database) in this stack is packaged as a docker container. Because of this, it can be deployed to any infrastructure capable of running containers. In this section we'll explain how to setup your own infrastructure where [AWS Fargate](https://aws.amazon.com/fargate/) will run your containers and setup a PostgreSQL database in [Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) that will hold your data. This infrastructure will be capable of supporting multiple applications and scale in time.

### Familiarize yourself with Fargate, ECS concepts and AWS[¶](#familiarize-yourself-with-fargate-ecs-concepts-and-aws "Permanent link")

*   [Fargate](https://aws.amazon.com/blogs/compute/aws-fargate-a-product-overview/)
*   [What is Amazon EC2 Container Service?](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
*   [Amazon EC2 Container Service](https://aws.amazon.com/ecs/)
*   [ECS - Getting Starterd](https://aws.amazon.com/ecs/getting-started/)
*   [Installing the AWS Command Line Interface](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

If you feel overwhelmed by all the AWS services and their names, check out [Amazon Web Services in Plain English](https://www.expeditedssl.com/aws-in-plain-english)

### Install AWS CLI[¶](#install-aws-cli "Permanent link")

We'll be interacting with AWS in some cases using it's command line interface. You will need to [install](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and [configure](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) it.

### Fargate Cluster[¶](#fargate-cluster "Permanent link")

Go to the [ECS Service](https://console.aws.amazon.com/ecs/home) page and click **Create Cluster** button and select **Networking only** template

![cluster-template](../../images/fargate/cluster-template.png)

Name the cluster **fargate-cluster** and choose to create a VPC for it.

![cluster-config](../../images/fargate/cluster-config.png)

Save the cluster name in a env var.

export CLUSTER\_NAME\=fargate-cluster

Save the stack name to env

export STACK\_NAME\=EC2ContainerService-$CLUSTER\_NAME

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
Cluster\_Param\_SubnetCidr2=10.10.1.0/24
Cluster\_Param\_EbsVolumeSize=0
...
...

Save the cluster region in a env var. First check where the cluster is located.

echo $Cluster\_Param\_VpcAvailabilityZones

save it to env

export AWS\_REGION\=us-east-1

### SSL Certificate[¶](#ssl-certificate "Permanent link")

If you want to use HTTPS, you will need a SSL certificate and you will need to complete this step before the next one (creating the loadbalancer). You can create (or upload) a certificate in [AWS Certificate Manager](http://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html). You must request (or configure) your certificate in the same region as your cluster, you can not use them across regions.

List the certificates

aws acm list-certificates

Save the ARN

export CERTIFICATE\_ARN\="arn:aws:acm:us-east-1:CHANGE-WITH-YOURS:certificate/CHANGE-WITH-YOURS"

### Loadbalancer[¶](#loadbalancer "Permanent link")

The loadbalancer will route traffic to our containers (just like the cluster, it can be used for multiple applications). If you do not need HTTPS, and you skipped the previous step, make sure you set `CERTIFICATE_ARN` to empty string

export CERTIFICATE\_ARN\=""

Download the cloudformation template that will create the loadbalancer

curl -SLO https://docs.subzero.cloud/cloudformation/loadbalancer.yml

Create the loadbalancer

aws cloudformation create-stack \\
    --stack-name $CLUSTER\_NAME\-loadbalancer \\
    --template-body file://loadbalancer.yml \\
    --capabilities CAPABILITY\_IAM \\
    --parameters \\
    ParameterKey\=ClusterName,ParameterValue\=$CLUSTER\_NAME \\
    ParameterKey\=CertificateArn,ParameterValue\=$CERTIFICATE\_ARN \\
    ParameterKey\=Vpc,ParameterValue\=$Cluster\_Resource\_Vpc \\
    ParameterKey\=PubSubnetAz1,ParameterValue\=$Cluster\_Resource\_PubSubnetAz1 \\
    ParameterKey\=PubSubnetAz2,ParameterValue\=$Cluster\_Resource\_PubSubnetAz2

### Image Repository[¶](#image-repository "Permanent link")

We'll store our OpenResty image (the only one that changes) in [Amazon EC2 Container Registry](https://aws.amazon.com/ecr/). You can use any docker image repository you like.

read project env vars

source .env

create the repository

aws ecr create-repository --repository-name $COMPOSE\_PROJECT\_NAME/openresty

extract the uri in a separate env var

export OPENRESTY\_REPO\_URI\=\`aws ecr describe-repositories --repository-name $COMPOSE\_PROJECT\_NAME/openresty --output text --query 'repositories\[0\].repositoryUri'\`

### Database (PostgreSQL in RDS)[¶](#database-postgresql-in-rds "Permanent link")

Go to the [RDS Service](https://console.aws.amazon.com/rds/home) page and click **Launch a DB Instance** button. Select **PostgreSQL** engine.

![db-engine.png](../../images/fargate/db-engine.png)

Select your use case (production/dev).

![db-usecase.png](../../images/fargate/db-usecase.png)

Choose the instance type options (start with a small size instance, like db.t2.small). Choose the **DB instance identifier** (ex. `subzerostarterkit-db`) and make sure to use the same identifier in the commands below. ![db-details-1.png](../../images/fargate/db-details-1.png) ![db-details-2.png](../../images/fargate/db-details-2.png)

Place the db in the same VPC as your cluster (check VPC Id using `echo $Cluster_Resource_Vpc`). Select _Public accessibility_ option.

![db-advanced-1.png](../../images/fargate/db-advanced-1.png)

Enter `app` for the _Database name_

![db-advanced-12.png](../../images/fargate/db-advanced-2.png)

Configure some firewall rules and make sure you can connect to the database.

get the database security group id

source .env && \\
export PRODUCTION\_DB\_INSTANCE\=$COMPOSE\_PROJECT\_NAME\-db && \\
export PRODUCTION\_DB\_SECURITY\_GROUP\=\`aws rds describe-db-instances --db-instance-identifier $PRODUCTION\_DB\_INSTANCE --output text --query 'DBInstances\[0\].VpcSecurityGroups\[0\].VpcSecurityGroupId'\`

allow Fargate tasks from our VPC to connect to this db

aws ec2 authorize-security-group-ingress \\
              --region $AWS\_REGION \\
              --group-id $PRODUCTION\_DB\_SECURITY\_GROUP \\
              --protocol tcp \\
              --port 5432 \\
              --cidr $Cluster\_Param\_SubnetCidr1

aws ec2 authorize-security-group-ingress \\
              --region $AWS\_REGION \\
              --group-id $PRODUCTION\_DB\_SECURITY\_GROUP \\
              --protocol tcp \\
              --port 5432 \\
              --cidr $Cluster\_Param\_SubnetCidr2

export production db host

export PRODUCTION\_DB\_HOST\=\`aws rds describe-db-instances --db-instance-identifier $PRODUCTION\_DB\_INSTANCE --output text --query 'DBInstances\[0\].Endpoint.Address'\`

Check the AWS management panel and wait until the database is "available". Check you can connect.

psql -h $PRODUCTION\_DB\_HOST -U $SUPER\_USER $DB\_NAME

Create the authenticator role used by PostgREST to connect.

psql \\
    -h $PRODUCTION\_DB\_HOST \\
    -U $SUPER\_USER \\
    -c "create role $DB\_USER with login password 'SET-YOUR-AUTHENTICATOR-PASSWORD';" $DB\_NAME

### Application Stack[¶](#application-stack "Permanent link")

Download the cloudformation template

curl -SLO https://docs.subzero.cloud/cloudformation/application.yml

Create the application cloudformation stack. Right now we will use `DesiredCount=0` since our application is not deployed yet (db is empty and the OpenResty images are not uploaded)

aws cloudformation create-stack \\
--stack-name $COMPOSE\_PROJECT\_NAME \\
--template-body file://application.yml \\
--capabilities CAPABILITY\_IAM \\
--parameters \\
ParameterKey\=ClusterName,ParameterValue\=$CLUSTER\_NAME \\
ParameterKey\=ClusterType,ParameterValue\=FARGATE \\
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

export SUBZERO\_APP\_CONF\=$(cat <<EOF
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
) && \\
echo "${SUBZERO\_APP\_CONF}" > subzero-app.json

### Deploy[¶](#deploy "Permanent link")

Make sure you have the database migrations folder ready for deploying it to the production RDS PostgreSQL instance (see [Managing Migrations](../managing-migrations/) for details).

Now for the last part of the deployment do:

subzero cloud app-deploy --dba $SUPER\_USER --password SET-YOUR-RDS-MASTER-PASSWORD-HERE

Sample output

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
  + 0000000001-initial .. ok

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
ParameterKey\=JwtSecret,UsePreviousValue\=true \\
