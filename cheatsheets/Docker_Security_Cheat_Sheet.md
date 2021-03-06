# Introduction

Docker containers are the most popular containerisation technology. Used properly can increase level of security (in comparison to running application directly on the host). On the other hand some misconfigurations can lead to downgrade level of security or even introduce new vulnerabilities.

The aim of this cheat sheet is to provide an easy to use list of common security mistakes and good practices that will help you securing your Docker containers.

# Rules

## RULE \#0 - Keep Host and Docker up to date

To prevent from known, container escapes vulnerabilities, which typically ends in escalating to root/administrator privileges, patching Docker Engine and Docker Machine is crucial.

In addition containers (unlike in a virtual machines) share kernel with the host, therefore kernel exploit runned inside the container will directly hit host kernel. For example kernel privilege escalation exploit ([like Dirty COW](https://github.com/scumjr/dirtycow-vdso)) runned inside well insulated container will result in root access in a host.

## RULE \#1 - Do not expose the Docker deamon socket (even to the containers)

Docker socket */var/run/docker.sock* is the UNIX socket that Docker is listening to. This is primary entry point for the Docker API. The owner of this socket is root. Giving someone access to it is equivalent to giving a unrestricted root access to your host. 

**Do not enable *tcp* Docker deamon socket.**  If you are running docker daemon with `-H tcp://0.0.0.0:XXX` or similar you are exposing un-encrypted and un-authenticated direct access to the Docker daemon. 
If you really, **really** have to do this you should secure it. Check how to do this [following Docker official documentation](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option).

**Do not expose */var/run/docker.sock* to other containers**. If you are running your docker image with `-v /var/run/docker.sock://var/run/docker.sock` or similar you should change it. Remember that mounting the socket read-only is not a solution but only makes it harder to exploit. Equivalent in docker-compose file is somethink like this:

```
volumes:
- "/var/run/docker.sock:/var/run/docker.sock"
```

## RULE \#2 - Set a user 

Configuring container, to use unprivileged user, is the best way to prevent privilege escalation attacks. This can be accomplished in three different ways:

1. During runtime using `-u` option of `docker run` command e.g.:

```
docker run -u 4000 alpine
```

2. During build time. Simple add user in Dockerfile and use it. For example:

```
FROM alpine
RUN groupadd -r myuser && useradd -r -g myuser myuser
<HERE DO WHAT YOU HAVE TO DO AS A ROOT USER LIKE INSTALLING PACKAGES ETC.>
USER myuser
```

3. Enable user namespace support (`--userns-remap=default`) in [Docker deamon](https://docs.docker.com/engine/security/userns-remap/#enable-userns-remap-on-the-daemon)

More informatrion about this topic can be found in [Docker official documentation](https://docs.docker.com/engine/security/userns-remap/)

## RULE \#3 - Limit capabilities (Grant only specific capabilities, needed by a container)

[Linux kernel capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) are set of privileges that can be used by privileged. Docker, by default, runs with only a  subset of capabilities. 
You can change it and drop some capabilities (using `--cap-drop`) to harden your docker containers, or add some capabilities (using `--cap-add`) if needed. 
Remember not to run containers with the `--privileged` flag - this will add ALL Linux kernel capabilities to the container. 

The most secure setup is to drop all capabilities `--cap-drop all` and then add only required ones. For example:

```
docker run --cap-drop all --cap-add CHOWN alpine
```

**And remember: Do not run containers with the *--privileged* flag!!!**

## RULE \#4 - Add –no-new-privileges flag

Always run your docker images with `--security-opt=no-new-privileges` in order to prevent escalate privileges using `setuid` or `setgid` binaries.

## RULE \#5 - Disable inter-container communication (--icc=false)

By default inter-container communication (icc) is enabled - it means that all containers can talk with each other (using [`docker0` bridged network](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/container-communication/#communication-between-containers)).
This can be disabled by running docker deamon with `--icc=false` flag. 
If icc is disabled (icc=false) it is required to tell which containers can communicate using --link=CONTAINER_NAME_or_ID:ALIAS option. 
See more in [Docker documentation - container communication](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/container-communication/#communication-between-containers)

## RULE \#6 - Use Linux Security Module (seccomp, AppArmor, or SELinux)

**First of all do not disable default security profile!** 

Consider using security profile like [seccomp](https://docs.docker.com/engine/security/seccomp/) or [AppArmor](https://docs.docker.com/engine/security/apparmor/). 

## RULE \#7 - Limit resources (memory, CPU, file descriptors, processes, restarts)

The best way to avoid DoS attacks is limiting resources. You can limit [memory](https://docs.docker.com/config/containers/resource_constraints/#memory), [CPU](https://docs.docker.com/config/containers/resource_constraints/#cpu), maximum number of restarts (`--restart=on-failure:<number_of_restarts>`), maximum number of file descriptors (`--ulimit nofile=<number>`) and maximum number of processes (`--ulimit nproc=<number>`).

[Check documentation for more details about ulimits](https://docs.docker.com/engine/reference/commandline/run/#set-ulimits-in-container---ulimit)

## RULE \#8 - Set filesystem and volumes to read-only 

**Run containers with a read-only filesystem** using `--read-only` flag. For example:

```
docker run --read-only alpine sh -c 'echo "whatever" > /tmp'
```

If application inside container have to save something temporarily combine `--read-only` flag with `--tmpfs` like this:

```
docker run --read-only --tmpfs /tmp alpine sh -c 'echo "whatever" > /tmp/file'
```

Equivalent in docker-compose file will be:

```
version: "3"
services:
  alpine:
    image: alpine
    read_only: true
```

In addition if volume is mounted only for reading **mount them as a read-only**
It can be done by appending `:ro` to the `-v` like this:

```
docker run -v volume-name:/path/in/container:ro alpine
```

Or by using `--mount` option:

```
$ docker run --mount source=volume-name,destination=/path/in/container,readonly alpine
```

## RULE \#9 - Use static analysis tools

To detect containers with known vulnerabilities - scan images using static analysis tools. 

- Free
  - [Clair](https://github.com/coreos/clair)
- Commercial
  - [Snyk](https://snyk.io/) **(open source and free option available)**
  - [anchore](https://anchore.com/opensource/) **(open source and free option available)**
  - [JFrog XRay](https://jfrog.com/xray/) 
  - [Qualys](https://www.qualys.com/apps/container-security/)

# Related Projects

[OWASP Docker Top 10](https://github.com/OWASP/Docker-Security)

# Authors and Primary Editors

Jakub Maćkowski - jakub.mackowski@owasp.org
