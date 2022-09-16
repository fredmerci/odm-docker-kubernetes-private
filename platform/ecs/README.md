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

### b. Create a secret by running a docker create secret command.

```console
docker secret create dockerhubAccessToken hyc.json
```

where:
* <API_KEY_GENERATED> is the entitlement key from the previous step. Make sure you enclose the key in double-quotes.
* <USER_EMAIL> is the email address associated with your IBMid.

> Note: The `cp.icr.io` value for the docker-server parameter is the only registry domain name that contains the images. You must set the docker-username to `cp` to use an entitlement key as docker-password.

The my-odm-docker-registry secret name is already used for the `image.pullSecrets` parameter when you run a helm install of your containers. The `image.repository` parameter is also set by default to `cp.icr.io/cp/cp4a/odm`.

## Run ODM container in ECS

### Use ECS Context
- Ensure you are using your ECS context. You can do this either by specifying
the `--context myecscontext` flag with your command, or by setting the
current context using the command `docker context use myecscontext`.
### 

## View application logs

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

# Useful links
   * Docker documentation : https://docs.docker.com/cloud/ecs-integration/
   *  If you have trouble to delete your stack read this documentation
https://aws.amazon.com/fr/premiumsupport/knowledge-center/cloudformation-stack-stuck-progress/