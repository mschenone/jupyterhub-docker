# jupyterhub-deploy-docker

## Quickstart
* Install docker
* Clone this repository
* Customize .env
* Setup authentication in jupyterhub_config.py
* Run make to create network, volumes and build JupyterHub image
* Build the JupyterLab image docker build -f Dockerfile.jupyterlab .
* docker-compose up -d


## Quickstart  by GGG

Testing and development
* Install docker
* Go inside a directory (e.g. /opt/)
* Clone this repository and cd inside (/opt/jupyterhub-deploy-docker)
* Create a "secrets" directory and put inside some empty file named "jupyterhub.key" and "jupyterhub.crt"
* Create a file called "userlist" which will be used to set admin flag to some users (see later in this readme)
* Go back to your base directory (e.g. /opt/) and clone [the notebook image](https://rlabgw0.unipv.it/tutorato_jupyterhub/tutorato-notebook-server)
* cd in tutorato-notebook-server
* Build the notebook image `docker build -t tutorato-notebook .`
* If no errors are shown, you can cd back into ../jupyterhub-deploy-docker
* jupyterhub_config.py [must be set with some form of authentication](https://github.com/jupyterhub/oauthenticator). Right now is set on our GitLab instance, but without actual secrets so it wont work.
* Run "make build" to build the JupyterHub image
* docker-compose up -d to run the container
* Cross finger. In case of error, use docker logs on the JupyterHub and/or on the notebook container, which should give some indication

Updating during development
* The C kernel is a jupyter machine itself, so per-debug can be built with `docker build -t test_kernel .` and started with `docker run -it -p 8888:8888 test_kernel`.
* Since running directly the jupyter machine might be useful, I had to fix permissions by adding a chmod 777 in the dockerscript, but in deployment this won't ever be launched anyway.
* Updates on the c kernel requires to stop jupyterhub and user containers, remove their containers, remove docker images for tutorato-notebook in order to force a git clone again.
* Updating the tutoratos sample should just require a jupyterhub reboot but will stop the users containers.

From testing to deploy:
* create a clean branch, commit files, push and open a merge request to update the GitLab repositories.
* Git clone on the server, similar to testing and deployment.
* configure https on the server. Use "true" certificates, obtained from letsencrypt. If possible, setup cron to renew them and restart jupyterhub like once a month.
* setup actual authentication using Google API console, again directly on the server. DO NOT PUSH SECRETS!
* Write documentation if neeeded.

The things below this quickstart are the original readme, which will require proper modifications BTW.

## Original readme

This repository provides a reference deployment of [JupyterHub](https://github.com/jupyter/jupyterhub), a multi-user [Jupyter Notebook](http://jupyter.org/) environment, on a **single host** using [Docker](https://docs.docker.com).  

This deployment:

* Runs the [JupyterHub components](https://jupyterhub.readthedocs.org/en/latest/getting-started.html#overview) in a Docker container on the host
* Uses [DockerSpawner](https://github.com/jupyter/dockerspawner) to spawn single-user Jupyter Notebook servers in separate Docker containers on the same host
* Persists JupyterHub data in a Docker volume on the host
* Persists user notebook directories in Docker volumes on the host
* Uses [OAuthenticator](https://github.com/jupyter/oauthenticator) and [Google Console OAuth](https://developers.google.com/identity/protocols/OAuth2) (GGG: TODO!) to authenticate users

![JupyterHub single host Docker deployment](internal/jupyterhub-docker.png)

## Use Cases

The tutorato in UniPV

## Disclaimer

This deployment is **NOT** intended for a production environment.  

## Prerequisites

* This deployment uses Docker for all the things, via  [Docker Compose](https://docs.docker.com/compose/overview/).
  It requires [Docker Engine](https://docs.docker.com/engine) 1.12.0 or higher.
  See the [installation instructions](https://docs.docker.com/engine/installation/) for your environment.
* This example configures JupyterHub for HTTPS connections (the default).
   As such, you must provide TLS certificate chain and key files to the JupyterHub server.
   If you do not have your own certificate chain and key, you can either
   [create self-signed versions](https://jupyter-notebook.readthedocs.org/en/latest/public_server.html#using-ssl-for-encrypted-communication),
   or obtain real ones from [Let's Encrypt](https://letsencrypt.org)
   (see the [letsencrypt example](examples/letsencrypt/README.md) for instructions).

From here on, we'll assume you are set up with docker,
via a local installation or [docker-machine](./docs/docker-machine.md).
At this point,

    docker ps

should work.


## Setup GitHub Authentication

This deployment uses GitHub OAuth to authenticate users.
It requires that you create a [Google oauth](https://developers.google.com/identity/protocols/OAuth2) (see developer console).
You will need to specify an OAuth callback URL in the following form:

```
https://<myhost.mydomain>/hub/oauth_callback
```

You must pass the secrets that Google provides for your application to JupyterHub at runtime.
You can do this by setting the `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`,
and `OAUTH_CALLBACK_URL` environment variables when you run the JupyterHub container,
or you can add them to the `.env` file in the root directory of this repository.  For example,

```
GOOGLE_CLIENT_ID=<github_client_id>
GOOGLE_CLIENT_SECRET=<github_client_secret>
GOOGLE_CALLBACK_URL=https://<myhost.mydomain>/hub/oauth_callback
```

**Note:** The `.env` file is a special file that Docker Compose uses to lookup environment variables.
If you choose to place the GitHub secrets in this file,
you should ensure that this file remains private
(e.g., do not commit the secrets to source control).

## Build the JupyterHub Docker image

Configure JupyterHub and build it into a Docker image.

1. Copy the TLS certificate chain and key files for the JupyterHub server to a directory named `secrets` within this repository directory. These will be added to the JupyterHub Docker image at build time.  If you do not have a certificate chain and key, you can either [create self-signed versions](https://jupyter-notebook.readthedocs.org/en/latest/public_server.html#using-ssl-for-encrypted-communication), or obtain real ones from [Let's Encrypt](https://letsencrypt.org) (see the [letsencrypt example](examples/letsencrypt/README.md) for instructions).

    ```
    mkdir -p secrets
    cp jupyterhub.crt jupyterhub.key secrets/
    ```

1. Create a `userlist` file with a list of authorized users.  At a minimum, this file should contain a single admin user.  The username should be a GitHub username.  For example:

   ```
   jtyberg admin
   ```

   The admin user will have the ability to add more users in the JupyterHub admin console.

1. Use [docker-compose](https://docs.docker.com/compose/reference/) to build the
   JupyterHub Docker image on the active Docker machine host:

    ```
    make build
    ```

## Prepare the Jupyter Notebook Image

You can configure JupyterHub to spawn Notebook servers from any Docker image, as
long as the image's `ENTRYPOINT` and/or `CMD` starts a single-user instance of
Jupyter Notebook server that is compatible with JupyterHub.

To specify which Notebook image to spawn for users, you set the value of the  
`DOCKER_NOTEBOOK_IMAGE` environment variable to the desired container image.
You can set this variable in the `.env` file, or alternatively, you can
override the value in this file by setting `DOCKER_NOTEBOOK_IMAGE` in the
environment where you launch JupyterHub.

Whether you build a custom Notebook image or pull an image from a public or
private Docker registry, the image must reside on the host.  

If the Notebook image does not exist on host, Docker will attempt to pull the
image the first time a user attempts to start his or her server.  In such cases,
JupyterHub may timeout if the image being pulled is large, so it is better to
pull the image to the host before running JupyterHub.  

This deployment defaults to the
[jupyter/scipy-notebook](https://hub.docker.com/r/jupyter/scipy-notebook/)
Notebook image, which is built from the `scipy-notebook`
[Docker stacks](https://github.com/jupyter/docker-stacks). (Note that the Docker
stacks `*-notebook` images tagged `2d878db5cbff` include the
`start-singleuser.sh` script required to start a single-user instance of the
Notebook server that is compatible with JupyterHub).

You can pull the image using the following command:

```
make notebook_image
```

## Run JupyterHub

Run the JupyterHub container on the host.

To run the JupyterHub container in detached mode:

```
docker-compose up -d
```

Once the container is running, you should be able to access the JupyterHub console at

```
https://myhost.mydomain
```

To bring down the JupyterHub container:

```
docker-compose down
```

## Behind the scenes

`make build` does a few things behind the scenes, to set up the environment for JupyterHub:

### Create a Docker Network

Create a Docker network for inter-container communication.  The benefits of using a Docker network are:

* container isolation - only the containers on the network can access one another
* name resolution - Docker daemon runs an embedded DNS server to provide automatic service discovery for containers connected to user-defined networks.  This allows us to access containers on the same network by name.

Here we create a Docker network named `jupyterhub-network`.  Later, we will configure the JupyterHub and single-user Jupyter Notebook containers to run attached to this network.

```
docker network create jupyterhub-network
```

### Create a JupyterHub Data Volume

Create a Docker volume to persist JupyterHub data.   This volume will reside on the host machine.  Using a volume allows user lists, cookies, etc., to persist across JupyterHub container restarts.

```
docker volume create --name jupyterhub-data
```

## FAQ

### How can I view the logs for JupyterHub or users' Notebook servers?

Use `docker logs <container>`.  For example, to view the logs of the `jupyterhub` container

```
docker logs jupyterhub
```

### How do I specify the Notebook server image to spawn for users?

In this deployment, JupyterHub uses DockerSpawner to spawn single-user
Notebook servers. You set the desired Notebook server image in a
`DOCKER_NOTEBOOK_IMAGE` environment variable.

JupyterHub reads the Notebook image name from `jupyterhub_config.py`, which
reads the Notebook image name from the `DOCKER_NOTEBOOK_IMAGE` environment
variable:

```
# DockerSpawner setting in jupyterhub_config.py
c.DockerSpawner.container_image = os.environ['DOCKER_NOTEBOOK_IMAGE']
```

By default, the`DOCKER_NOTEBOOK_IMAGE` environment variable is set in the
`.env` file.

```
# Setting in the .env file
DOCKER_NOTEBOOK_IMAGE=jupyter/scipy-notebook:2d878db5cbff
```

To use a different notebook server image, you can either change the desired
container image value in the `.env` file, or you can override it
by setting the `DOCKER_NOTEBOOK_IMAGE` variable to a different Notebook
image in the environment where you launch JupyterHub. For example, the
following setting would be used to spawn single-user `pyspark` notebook servers:

```
export DOCKER_NOTEBOOK_IMAGE=jupyterhub/pyspark-notebook:2d878db5cbff

docker-compose up -d
```

### If I change the name of the Notebook server image to spawn, do I need to restart JupyterHub?

Yes. JupyterHub reads its configuration which includes the container image
name for DockerSpawner. JupyterHub uses this configuration to determine the
Notebook server image to spawn during startup.

If you change DockerSpawner's name of the Docker image to spawn, you will
need to restart the JupyterHub container for changes to occur.

In this reference deployment, cookies are persisted to a Docker volume on the
Hub's host. Restarting JupyterHub might cause a temporary blip in user
service as the JupyterHub container restarts. Users will not have to login
again to their individual notebook servers. However, users may need to
refresh their browser to re-establish connections to the running Notebook
kernels.

### How can I backup a user's notebook directory?

There are multiple ways to [backup and restore](https://docs.docker.com/engine/userguide/containers/dockervolumes/#backup-restore-or-migrate-data-volumes) data in Docker containers.  

Suppose you have the following running containers:

```
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}"

CONTAINER ID        IMAGE                    NAMES
bc02dd6bb91b        jupyter/minimal-notebook jupyter-jtyberg
7b48a0b33389        jupyterhub               jupyterhub
```

In this deployment, the user's notebook directories (`/home/jovyan/work`) are backed by Docker volumes.

```
docker inspect -f '{{ .Mounts }}' jupyter-jtyberg

[{jtyberg /var/lib/docker/volumes/jtyberg/_data /home/jovyan/work local rw true rprivate}]
```

We can backup the user's notebook directory by running a separate container that mounts the user's volume and creates a tarball of the directory.  

```
docker run --rm \
  -u root \
  -v /tmp:/backups \
  -v jtyberg:/notebooks \
  jupyter/minimal-notebook \
  tar cvf /backups/jtyberg-backup.tar /notebooks
```

The above command creates a tarball in the `/tmp` directory on the host.
