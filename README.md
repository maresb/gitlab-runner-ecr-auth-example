Preliminary instructions:

# Prerequisite helpers

```
cd build-amazon-ecr-credential-helpers
./do-builds
ls docker-credential-ecr-login-*  # Should see two files here.
cd ..
```

# Runner-only config

```
docker-compose up -d  # Leave out -d to see output for debugging.
./register  # Follow the instructions from server for adding a runner.
```

# Runner + Server config

Add the line
```
127.0.0.1    gitlab-server
```
to `/etc/hosts` so that the server can be properly resolved from the host.

```
docker-compose up -f docker-compose-complete.yaml -d
./register-complete  # Follow the instructions from server for adding a runner.
```
