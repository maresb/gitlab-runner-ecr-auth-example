The `docker-credentials-ecr-login` binary is used by Docker to
automatically obtain temporary login credentials. In order to be
able to use it with prebuilt images, we can precompiling it,
store the binary on the host, and bind mount it to the desired
containers.

The two images we will use are
[`gitlab/gitlab-runner`](https://hub.docker.com/r/gitlab/gitlab-runner/)
and [`docker`](https://hub.docker.com/_/docker), where
`gitlab/gitlab-runner` is an Ubuntu-based image, while `docker` is an
Alpine-based image.

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

As Eisfunke explains [in this forum post](https://forum.gitlab.com/t/bin-sh-eval-line-97-mybinary-not-found/27125/3
), the problem is that the correct linked libraries cannot be found.

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

The solution is to build separate binaries. To do this, simply execute the `do-builds` script.
