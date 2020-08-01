Here we build Amazon's `docker-credentials-ecr-login` from the
[Amazon ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper)
GitHub repository.

# The Solution

As explained below, we need to build a `docker-credentials-ecr-login`
binary for each Docker image we want to use. This is automated in the
`do-builds` script. Thus it is recommended to simply run
```
./do-builds
```

In case you would like to build the binary for a different image,
you can adapt the following command
```
./build alpine docker:19.03.12 docker-credential-ecr-login-docker_19.03.12
```

which uses the Alpine-based build script in `Dockerfile.alpine` starting
from the `docker:19.03.12` image, and saves the resulting binary as
`docker-credential-ecr-login-docker_19.03.12`.

# The Problem

The `docker-credentials-ecr-login` binary is used by Docker to
automatically obtain temporary login credentials. We need to use
it with the Docker images
[`gitlab/gitlab-runner`](https://hub.docker.com/r/gitlab/gitlab-runner/)
and [`docker`](https://hub.docker.com/_/docker).

To inject this binary into the images, we can precompile it,
store the binary on the host, and bind mount it to the corresponding
containers.

If one tries to use a `docker-credentials-ecr-login` binary built in one
image with the other image, one encounters a mysterious error message
such as

```
sh: 1: docker-credentials-ecr-login: not found
```

or

```
bash: docker-credentials-ecr-login: No such file or directory
```

even when the binary is in the `$PATH` and has the correct executable
permissions.

As Eisfunke explains 
[in this forum post](https://forum.gitlab.com/t/bin-sh-eval-line-97-mybinary-not-found/27125/3
), the problem is that the correct linked libraries cannot be found.
This is perhaps not so surprising since `gitlab/gitlab-runner` is an
Ubuntu-based image, while `docker` is an Alpine-based image.

Indeed, checking the linked libraries, we see

```
$ docker run --rm -it \
    -v $PWD/docker-credential-ecr-login-docker_19.03.12:/usr/local/bin/docker-credential-ecr-login \
    --entrypoint /bin/bash \
    gitlab/gitlab-runner:v13.2.1 \
    ldd /usr/local/bin/docker-credential-ecr-login \
;
	linux-vdso.so.1 (0x00007ffd767bf000)
	libc.musl-x86_64.so.1 => not found	linux-vdso.so.1 (0x00007ffd9774a000)
	libc.musl-x86_64.so.1 => not found
```

```
$ docker run --rm -it \
    -v $PWD/docker-credential-ecr-login-gitlab-runner_v13.2.1:/usr/local/bin/docker-credential-ecr-login \
    docker:19.03.12 \
    ldd /usr/local/bin/docker-credential-ecr-login \
;
	/lib64/ld-linux-x86-64.so.2 (0x7fd1a1634000)
	libpthread.so.0 => /lib64/ld-linux-x86-64.so.2 (0x7fd1a1634000)
	libc.so.6 => /lib64/ld-linux-x86-64.so.2 (0x7fd1a1634000)
Error relocating /usr/local/bin/docker-credential-ecr-login: __vfprintf_chk: symbol not found
Error relocating /usr/local/bin/docker-credential-ecr-login: __fprintf_chk: symbol not found
```

In order to be sure that `docker-credentials-ecr-login` works with
various images, we build a separate binary for each individual
image.
