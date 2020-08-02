# gitlab-runner-ecr-auth-example

* GitHub: <https://github.com/maresb/gitlab-runner-ecr-auth-example>
* GitLab: <https://gitlab.com/bmares/gitlab-runner-ecr-auth-example>

## Introduction

Here is a complete Docker-Compose-based example for how to configure
a GitLab runner to work with automatic ECR authentication. This allows
for access to private ECR registries for
both the Docker executor and Docker-in-Docker (DinD).

These instructions make the following assumptions.

* You are using Linux. (If not, you deserve better.)
* Your user has
[permissions to run `docker` without `sudo`](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user).
* You are entirely responsible for all aspects of security. This setup is meant
only for example purposes where all users are trusted. For more information, see [Security](#security).

## Quick start

In case you just want to understand how this works, skip to the section [The authentication mechanism](#the-authentication-mechanism).

If you have an existing GitLab server, you should do the [prerequisites](#prerequisites)
and then use the [runner-only configuration](#runner-only-configuration).

If you have no existing GitLab server, you can easily run a GitLab
server container. After doing the [prerequisites](#prerequisites)
you should skip to the [runner with server configuration](#runner-with-server-configuration).

### Prerequisites

Regardless of whether or not we need to run a server, we must configure the credentials and build the credential helpers.

#### Configure the credentials

* Copy the file `runner-ecr-credentials` to some relatively secure location
on the host (preferably outside of any Git repository where the credentials could
be accidentally committed and pushed).

* Edit the `.env` file and set the `RUNNER_CREDENTIALS_FILE` variable to the
corresponding path and filename.

#### Build the credential helpers

For more details, see [the corresponding README](build-amazon-ecr-credential-helpers/README.md).

``` bash
cd build-amazon-ecr-credential-helpers
./do-builds
ls docker-credential-ecr-login-*  # Should see two files here.
cd ..
```

### Runner-only configuration

If you also need to set up a GitLab server, then skip to the section
[Runner with Server configuration](#runner-with-server-configuration).

At this point you should have completed the [prerequisites](#prerequisites).

From the root of the `gitlab-runner-ecr-auth-example/` directory, run

``` bash
docker-compose up -d
./register  # Follow the instructions from server for adding a runner.
```

Logs can then be monitored with a command such as

``` bash
docker-compose logs -f --tail=50
```

(Before the runner is registered, it is normal to see error messages in
the logs about the missing `config.toml` file.)

After registration, your configured credentials should work automatically.

If desired,
[configure the restart behavior](https://docs.docker.com/compose/compose-file/#restart)
for the runner in `docker-compose.yaml`.

### Runner with Server configuration

At this point you should have completed the [prerequisites](#prerequisites).

We will set up the local GitLab server with the hostname `gitlab-server`.
In order that it is locally accessible from this hostname, we must add the
line

``` text
127.0.0.1    gitlab-server
```

to `/etc/hosts` so that the server can be properly resolved from the host.

The configuration which includes both the GitLab Runner and GitLab Server
is given in the `docker-compose-complete.yaml` file. Thus all `docker-compose`
commands (including `docker-compose down`) must be adjusted accordingly by
adding `-f docker-compose-complete.yaml` immediately after `docker-compose`.

To start the runner and server, run

``` bash
docker-compose -f docker-compose-complete.yaml up -d
```

Logs can be monitored with a command such as

``` bash
docker-compose -f docker-compose-complete.yaml logs -f --tail=50
```

(Before the runner is registered, it is normal to see error messages in
the logs about the missing `config.toml` file.)

It can take several minutes for the GitLab server to start.
When ready, the server will become available on
<http://gitlab-server:8800>.

Once it starts, you can configure a root account, and create a new project.
To register a runner, with this project, you should run the
`register-complete` script. (The only difference with the `register`
script is that `register-complete` configures the runner to use the same
Docker network as the GitLab server.)

``` bash
./register-complete  # Follow the instructions from server for adding a runner.
```

If desired, [configure the restart behavior](https://docs.docker.com/compose/compose-file/#restart)
for the runner and server in `docker-compose-complete.yaml`.

### Cleaning up

To stop the runner, use

``` bash
docker-compose down
```

To stop both the server and the runner, use

``` bash
docker-compose -f docker-compose-complete.yaml down
```

To reclaim disk space, most will be taken up by the GitLab Docker images. These can be
removed by running

``` bash
docker rmi gitlab/gitlab-runner:v13.2.1
docker rmi gitlab/gitlab-ce:latest
```

Additional files and directories which are created within this repository include:

* the binaries for the credential helper in `build-amazon-ecr-credential-helpers/`,
* the runner configuration in `gitlab-runner1-config/config.toml`,
* additional shared runner folders under `gitlab-runner1-mounts/`,
* in the case of the server, shared folders under `gitlab-server-mounts/`.

## Explanation

### Two distinct authentication requirements

There are two completely separate (but easily confusable) processes which must be separately configured.

#### Authentication for the runner's Docker executor

Suppose have a job that should be run in a private image, for instance

``` yaml
test-runner-pull:
  image: $PRIVATE_ECR_IMAGE
  script:
    - echo "Hello World!"
```

In this case, the runner's Docker executor must first obtain credentials
so that `$PRIVATE_ECR_IMAGE` can be pulled.

#### Authentication for running `docker` in a CI script

Suppose we have a script which uses the `docker` command to push or
pull from a private ECR registry, for instance

``` yaml
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

### The authentication mechanism

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

``` text
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

* privileged containers are not allowed (`docker-compose.yaml`), but `/var/run/docker.sock` is bind-mounted (`docker-compose.yaml`),
* `/root/.aws/credentials` is bind-mounted (`docker-compose.yaml`),
* `/usr/local/bin/docker-credential-ecr-login` is bind-mounted (`docker-compose.yaml`),
* `DOCKER_AUTH_CONFIG` is set as an environment variable (`register`/`config.toml`).

## Security

This configuration allows privileged containers to be run from the runner host.
Also, the AWS credentials are stored unencrypted on the host filesystem, and bound
to all running containers. Therefore, access to the runner should only be granted
to people who are both

* trusted to with the AWS credentials and
* trusted as administrators of the runner host.

I make absolutely no claims as to the security of this configuration, and assume that
you know what you are doing.
