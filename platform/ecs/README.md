# 1. Pre-requisite 

To deploy Docker containers on ECS, you must meet the following requirements:

   * Download and install the latest version of Docker Desktop.
      * Download for Mac
      * Download for Windows

Alternatively, install the Docker Compose CLI for Linux.

   * Ensure you have an [AWS Account](https://aws.amazon.com/getting-started/).

Docker not only runs multi-container applications locally, but also enables developers to seamlessly deploy Docker containers on Amazon ECS using a Compose file with the docker compose up command. The following sections contain instructions on how to deploy your Compose application on Amazon ECS.

# 2. Prepare your environment for the ODM installation (10 min)
## Create an AWS context
Once you have setup your AWS account you should create a docker context.

```
docker context create ecs myecscontext
```

This command create an Amazon ECS Docker context named myecscontext. If you have already installed and configured the AWS CLI, the setup command lets you select an existing AWS profile to connect to Amazon. Otherwise, you can create a new profile by passing an AWS access key ID and a secret access key. Finally, you can configure your ECS context to retrieve AWS credentials by AWS_* environment variables, which is a common way to integrate with third-party tools and single-sign-on providers.
```console
? Create a Docker context using:  [Use arrows to move, type to filter]
>  An existing AWS profile
  AWS secret and token credentials
  AWS environment variables
```

After you have created an AWS context, you can list your Docker contexts by running the `docker context ls` command:
```console
NAME                TYPE                DESCRIPTION                               DOCKER ENDPOINT               KUBERNETES ENDPOINT   ORCHESTRATOR
myecscontext        ecs                 credentials read from environment
default *           moby                Current DOCKER_HOST based configuration   unix:///var/run/docker.sock                         swarm
```

## Create Secret for the Entitled registry
To get access to the ODM material, you must have an IBM entitlement registry key to pull the images from the IBM Entitled registry. 
It's what will be used in the next step of this tutorial.

### a. Retrieve your entitled registry key
  - Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.

  - In the Container software library tile, verify your entitlement on the View library page, and then go to *Get entitlement key* to retrieve the key.

### b. Create a docker secret file 

Create a `token.json` file with that format.
```json
{
    "username":"cp",
    "password":"<YOUR_ENTITLED_API_KEY"
}
```

### b. Create a secret by running a docker create secret command.

You can then create a secret from this file using `docker secret`:

```console
$ docker secret create ICRAccessToken token.json
arn:aws:secretsmanager:eu-west-3:675801125365:secret:ICRAccessToken-XXXX
```
Once created, you can use this ARN in your Compose file using `x-aws-pull_credentials` custom extension with the Docker image URI for your service.

### c. Create a .env file
With the ARN generated previously create a .env file the ICRPULLSECRET variable.
```console
echo "ICRPULLSECRET=arn:aws:secretsmanager:eu-west-3:675801125365:secret:ICRAccessToken-XXXX" > .env
```
## Run ODM container in ECS

### Create docker-compose file
```yaml
services:
  dbserver:
    image: cp.icr.io/cp/cp4a/odm/dbserver:8.11.0.1-amd64
    x-aws-pull_credentials: "${ICRPULLSECRET}" 
    user: "26:26"
    ports:
    - 5432:5432
    environment:
      - POSTGRESQL_USER=odmusr
      - POSTGRESQL_PASSWORD=odmpwd
      - POSTGRESQL_DB=odmdb
      - SAMPLE=true
      # Should be removed for postgres UBI RHEL Based image.
#      - PGUSER=odmusr
      - PGDATA=/var/lib/postgresql/data
#      - SAMPLE=true
# Uncomment this line to persist your data. Note that on OSX you need to share this
# current directory in the Preference menu -> File Sharing menu.
#    volumes:
#      - ./pgdata:/pgdata


  odm-decisionserverconsole:
    image: cp.icr.io/cp/cp4a/odm/odm-decisionserverconsole:8.11.0.1-amd64
    depends_on:
    - dbserver
    ports:
    - 9853
    environment:
      - USERS_PASSWORD=odmAdmin
      - HTTPS_PORT=9853
    x-aws-pull_credentials:  "${ICRPULLSECRET}" 
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 512M


  odm-decisionrunner:
    image: cp.icr.io/cp/cp4a/odm/odm-decisionrunner:8.11.0.1-amd64
    depends_on:
    - dbserver
    - odm-decisionserverconsole
    environment:
      - HTTPS_PORT=9753
    x-aws-pull_credentials: "${ICRPULLSECRET}" 
    ports:
    - 9753
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 512M

  odm-decisionserverruntime:
    image: cp.icr.io/cp/cp4a/odm/odm-decisionserverruntime:8.11.0.1-amd64
    environment:
      - DECISIONSERVERCONSOLE_NAME=odm-decisionserverconsole
      - HTTPS_PORT=9953
    x-aws-pull_credentials: "${ICRPULLSECRET}" 
    depends_on:
    - dbserver
    - odm-decisionserverconsole
    ports:
    - 9953
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 512M

  odm-decisioncenter:
    image: cp.icr.io/cp/cp4a/odm/odm-decisioncenter:8.11.0.1-amd64
    depends_on:
    - dbserver
    environment:
    - DECISIONSERVERCONSOLE_PORT=9853
    - DECISIONRUNNER_PORT=9753
    - HTTPS_PORT=9653
    x-aws-pull_credentials: "${ICRPULLSECRET}" 
    ports:
    - 9653
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G
```
Save this content in a `docker-compose.yaml` file.

### Switch your docker environment to the ECS Context
- Ensure you are using your ECS context. You can do this either by specifying
the `--context myecscontext` flag with your command, or by setting the
current context using the command 
```console
docker context use myecscontext
```

> Note if you want to restore your initial docker environment `docker context use default`

### Run the ODM on ECS topology
```console
docker compose up

[+] Running 31/33
 ⠴ ecs                                            CreateInProgress User Initiated                                                                                                                                                                                       75.5s
 ⠿ DefaultNetwork                                 CreateComplete                                                                                                                                                                                                         6.0s
 ⠿ DbserverTCP5432TargetGroup                     CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ CloudMap                                       CreateComplete                                                                                                                                                                                                        46.0s
 ⠿ Cluster                                        CreateComplete                                                                                                                                                                                                         5.0s
 ⠿ OdmdecisioncenterTCP9653TargetGroup            CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ OdmdecisioncenterTaskExecutionRole             CreateComplete                                                                                                                                                                                                        26.0s
 ⠿ OdmdecisionrunnerTaskExecutionRole             CreateComplete                                                                                                                                                                                                        23.0s
 ⠿ LogGroup                                       CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ OdmdecisionserverconsoleTCP9853TargetGroup     CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ OdmdecisionserverconsoleTaskExecutionRole      CreateComplete                                                                                                                                                                                                        24.0s
 ⠴ LoadBalancer                                   CreateInProgress Resource creation Initiated                                                                                                                                                                          69.5s
 ⠿ EcsmycertSecret                                CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ OdmdecisionrunnerTCP9753TargetGroup            CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ OdmdecisionserverruntimeTCP9953TargetGroup     CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ DbserverTaskExecutionRole                      CreateComplete                                                                                                                                                                                                        24.0s
 ⠿ OdmdecisionserverruntimeTaskExecutionRole      CreateComplete                                                                                                                                                                                                        24.0s
 ⠿ Default9653Ingress                             CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ Default9953Ingress                             CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ Default5432Ingress                             CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ DefaultNetworkIngress                          CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ Default9853Ingress                             CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ Default9753Ingress                             CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ OdmdecisionrunnerTaskDefinition                CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ OdmdecisionserverconsoleTaskDefinition         CreateComplete                                                                                                                                                                                                         3.0s
 ⠿ DbserverTaskDefinition                         CreateComplete                                                                                                                                                                                                         3.0s
 ⠿ OdmdecisioncenterTaskDefinition                CreateComplete                                                                                                                                                                                                         4.0s
 ⠿ OdmdecisionserverruntimeTaskDefinition         CreateComplete                                                                                                                                                                                                         4.0s
 ⠿ OdmdecisioncenterServiceDiscoveryEntry         CreateComplete                                                                                                                                                                                                         1.0s
 ⠿ OdmdecisionserverruntimeServiceDiscoveryEntry  CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ DbserverServiceDiscoveryEntry                  CreateComplete                                                                                                                                                                                                         5.0s
 ⠿ OdmdecisionserverconsoleServiceDiscoveryEntry  CreateComplete                                                                                                                                                                                                         2.0s
 ⠿ OdmdecisionrunnerServiceDiscoveryEntry         CreateComplete
```


After a couple of minutes your ODM containers topology should be avalaible.

You can check your container is running by using 
```
docker compose ps

NAME                                        COMMAND             SERVICE                     STATUS              PORTS
task/ecs/50c5ec5d176b4c96b5a62dc03d8b8bf7   ""                  odm-decisionrunner          Activating          ecs-LoadBal-QMPILXMWO6F3-1d82c8ee22304c71.elb.eu-west-3.amazonaws.com:9753:9753->9753/tcp
task/ecs/8cb6f85ef9814055ae8308b9c7dc0f2b   ""                  odm-decisionserverconsole   Running             ecs-LoadBal-QMPILXMWO6F3-1d82c8ee22304c71.elb.eu-west-3.amazonaws.com:9853:9853->9853/tcp
task/ecs/93789440198b4b3eab4941273ae7905c   ""                  odm-decisionserverruntime   Pending             ecs-LoadBal-QMPILXMWO6F3-1d82c8ee22304c71.elb.eu-west-3.amazonaws.com:9953:9953->9953/tcp
task/ecs/cf749aa0d2a74966a22e377e3685daef   ""                  dbserver                    Running             ecs-LoadBal-QMPILXMWO6F3-1d82c8ee22304c71.elb.eu-west-3.amazonaws.com:5432:5432->5432/tcp
task/ecs/ebc1606437034c65a479f7dae101618a   ""                  odm-decisioncenter          Running             ecs-LoadBal-QMPILXMWO6F3-1d82c8ee22304c71.elb.eu-west-3.amazonaws.com:9653:9653->9653/tcp
```
When the status is Running you can access the containers by using the `Ports` urls.

### View application logs

The Docker Compose CLI configures AWS CloudWatch Logs service for your
containers.
By default you can see logs of your compose application the same way you check logs of local deployments:

```console
# fetch logs for application in current working directory
$ docker compose logs

# specify compose project name
$ docker compose --project-name PROJECT logs

# specify compose file
$ docker compose --file /path/to/docker-compose.yaml logs
```

A log group is created for the application as `docker-compose/<application_name>`,
and log streams are created for each service and container in your application
as `<application_name>/<service_name>/<container_ID>`.

You can fine tune AWS CloudWatch Logs using extension field `x-aws-logs_retention`
in your Compose file to set the number of retention days for log events. The
default behavior is to keep logs forever.

You can also pass `awslogs`
parameters to your container as standard
Compose file `logging.driver_opts` elements. See [AWS documentation](https://docs.amazonaws.cn/en_us/AmazonECS/latest/developerguide/using_awslogs.html){:target="_blank" rel="noopener" class="_"} for details on available log driver options.

## Useful links
   * Docker documentation : https://docs.docker.com/cloud/ecs-integration/
   *  If you have trouble to delete your stack read this documentation
https://aws.amazon.com/fr/premiumsupport/knowledge-center/cloudformation-stack-stuck-progress/

## TODO
- [ ] Bench 
- [ ] Document customization
- [ ] Document Replace DBServer by an RDS Database
- [ ] Add an architecture diagram
- [ ] Give a customization with an OIDC Provider
