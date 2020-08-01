# Introduction

Here is a complete Docker-Compose-based example for how to configure
a GitLab runner to work with automatic ECR authentication. This allows
for access to private ECR registries for
both the Docker executor and Docker-in-Docker (DinD).

These instructions assume that you are using Linux, and that your user has
[permissions to run `docker` without `sudo`](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user).

If you are not running Linux, you deserve better.

# Quick start

If you have an existing GitLab server, you should do the [prerequisites](#prerequisites)
and then use the [runner-only configuration](#runner-only-configuration).

If you have no existing GitLab server, you can easily run a GitLab
server container. After doing the [prerequisites](#prerequisites)
you should skip to the [runner and server configuration](#runner-and-server-configuration).

## Prerequisites

Regardless of whether or not we need to run a server, we must configure the credentials and build the credential helpers.

### Configure the credentials

* Copy the file `runner-ecr-credentials` to some relatively secure location
on the host (preferably outside of any Git repository where the credentials might
be accidentally committed and pushed). 

* Edit the `.env` file and set the `RUNNER_CREDENTIALS_FILE` variable to the
corresponding path and filename.

### Build the credential helpers

For more details, see [the corresponding README](build-amazon-ecr-credential-helpers/README.md).

```
cd build-amazon-ecr-credential-helpers
./do-builds
ls docker-credential-ecr-login-*  # Should see two files here.
cd ..
```

## Runner-only configuration

From the root of the `gitlab-runner-ecr-auth-example/` directory, run

```
docker-compose up -d
./register  # Follow the instructions from server for adding a runner.
```

Logs can then be monitored with a command such as
```
docker-compose logs -f --tail=50
```
After registration, your configured credentials should work automatically.

## Runner and Server configuration

We will set up the local GitLab server with the hostname `gitlab-server`.
In order that it is locally accessible from this address, we must add the
line
```
127.0.0.1    gitlab-server
```
to `/etc/hosts` so that the server can be properly resolved from the host.

The configuration which includes both the GitLab Runner and GitLab Server
is given in the `docker-compose-complete.yaml` file.

Run
```
docker-compose -f docker-compose-complete.yaml up -d
```
Logs can be monitored with a command such as
```
docker-compose -f docker-compose-complete.yaml logs -f --tail=50
```
When ready, the server will become available on http://gitlab-server:8800.


It is normal to see error messages after running `docker-compose up` but before
registering the runner, complaining that `config.toml` does not exist.

It can take several minutes for the GitLab server to start.

Once it starts, you can configure a root account and get it ready to register
a runner.

The corresponding `register-complete` script is identical to `register`
except that the runner is configured to launch containers
on the same Docker network as the GitLab server.

```
./register-complete  # Follow the instructions from server for adding a runner.
```


# Explanation

## Two distinct authentication requirements

There are two completely separate (but easily confusable) processes which must be separately configured.

### Authentication for the runner's Docker executor

Suppose have a job that should be run in a private image, for instance
```
test-runner-pull:
  image: $PRIVATE_ECR_IMAGE
  script:
    - echo "Hello World!"
```
In this case, the runner's Docker executor must first obtain credentials
so that `$PRIVATE_ECR_IMAGE` can be pulled.

### Authentication for running `docker` in a CI script

Suppose we have a script which uses the `docker` command to push or
pull from a private ECR registry, for instance

```
# dind service required for test-dind
services:
  - docker:19.03.12-dind

test-dind:
  image: docker:19.03.12
  script:
    - export
    - docker pull $PRIVATE_ECR_IMAGE
```

The `docker:19.03.12-dind` service provides a Docker host which can then be
accessed via the `docker` command from the `docker:19.03.12` image. To obtain
credentials, one could install and use the AWS CLI. However, this requires
installing Python as a dependency, and it clutters the CI script with a lot
of boilerplate code. Thus it's more desirable when authentication happens
transparently.

## The authentication mechanism

To handle authentication, we add a bind mount to the credential helper
under `/usr/local/bin/docker-credentials-ecr-login`.
Next we must signal to Docker that it should actually use it.
For the GitLab runner's Docker executor, this is done
by setting the `DOCKER_AUTH_CONFIG` environment variable to 
`{ "credsStore": "ecr-login" }`. For DinD (using the
`docker` command in a script), we must put the same content in
`/root/.docker/config.json` (which we accomplish with a bind mount).

Like the AWS CLI, `docker-credentials-ecr-login` will search for
credentials either in the default profile of the
`~/.aws/credentials` or in the following environment variables.
```
AWS_DEFAULT_REGION
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```
We take the former approach and mount a credentials file to
`/root/.aws/credentials`.

For DinD, everything is configured in `config.toml` via the `register`
command, namely:
* privileged containers are allowed (`register`/`config.toml`),
* `/root/.aws/credentials` is bind-mounted (`register`/`config.toml`),
* `/usr/local/bin/docker-credential-ecr-login` is bind-mounted (`register`/`config.toml`), and
* `/root/.docker/config.json` is bind-mounted (`register`/`config.toml`).

For the GitLab runner,
* privileged containers are not allowed (`docker-compose.yaml`)
* but `/var/run/docker.sock` is bind-mounted (`docker-compose.yaml`),
* `/root/.aws/credentials` is bind-mounted (`docker-compose.yaml`),
* `/usr/local/bin/docker-credential-ecr-login` is bind-mounted (`docker-compose.yaml`),
* `DOCKER_AUTH_CONFIG` is set as an environment variable (`register`/`config.toml`).
