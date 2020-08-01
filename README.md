# Introduction

Here is a complete Docker-Compose-based example for how to configure
a GitLab runner to work with automatic ECR authentication. This allows
for access to private ECR registries for
both the Docker executor and Docker-in-Docker (DinD).

These instructions assume that you are using Linux. If you are not running Linux,
you deserve better.

# Quick start

If we have an existing GitLab server, we can use the runner-only configuration.

If we have no existing GitLab server, we can easily run a GitLab
server container.

In both cases, we must configure the credentials and build the credential helpers.

## Prerequisites

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
docker-compose up -d  # Leave out -d to see output for debugging.
./register  # Follow the instructions from server for adding a runner.
```

It is normal to see error messages between running `docker-compose up` and `register`
complaining that `config.toml` does not exist.

Now if you properly assign your runner, it should automatically authenticate.

## Runner and Server configuration

We will set up the local GitLab server with the hostname `gitlab-server`.
In order that it is locally accessible from this address, we must add the
line
```
127.0.0.1    gitlab-server
```
to `/etc/hosts` so that the server can be properly resolved from the host.

With `docker-compose` We must reference `docker-compose-complete.yaml` since it contains the additional
configuration for the server.

Run
```
docker-compose up -f docker-compose-complete.yaml -d
```

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
of boilerplate code.
