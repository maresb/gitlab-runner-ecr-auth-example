# Introduction

Here is a complete Docker-Compose-based example for how to configure
a GitLab runner to work with automatic ECR authentication. This allows
for access to private ECR registries for
both the Docker executor and Docker-in-Docker (DinD).

# Quick start

If we have an existing GitLab server, we can use the runner-only configuration.

If we have no existing GitLab server, with minor modifications we can easily run a GitLab
server container.

In both cases, we must first build the credential helpers.

## Prerequisite: building the credential helpers

```
cd build-amazon-ecr-credential-helpers
./do-builds
ls docker-credential-ecr-login-*  # Should see two files here.
cd ..
```


## Runner-only configuration

```
docker-compose up -d  # Leave out -d to see output for debugging.
./register  # Follow the instructions from server for adding a runner.
```

## Runner and Server configuration


Add the line
```
127.0.0.1    gitlab-server
```
to `/etc/hosts` so that the server can be properly resolved from the host.

```
docker-compose up -f docker-compose-complete.yaml -d
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
