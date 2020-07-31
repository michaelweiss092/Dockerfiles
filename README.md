# Package CrowdStrike Falcon Linux sensor as a container

## Pre-reqs

* Build user mode, kernel modules and KMA bits for the respective build type.
  They'll be picked up by container packaging.
* Install `docker` and `ldd` if not already present on the build host.

## Build

* Build the container using the included ``Dockerfile``:
  * ``$ docker build --no-cache=true --build-arg \
      BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
      VCS_REF=$(git rev-parse --short HEAD) \
      -t falcon-sensor:latest .``

## Run

`falcon-sensor` is the default command of the container.  It can be invoked
as follows:

```
docker run --rm -ti --privileged --net=host -v /var/log:/var/log falcon-sensor
```

This is like running falcon-sensor directly on the console.  Standard terminal
output appears.  Pressing Control-C would cause `docker` client to pass
`SIGINT` along to the sensor, which would then exit.

`--net=host` is required for the containerized sensor to talk to the kernel
module over netlink and to Cloudsim over localhost.  Host `pid`, `uts` and
`ipc` namespaces are passed through to sensor container for easy access to
host resources.  Following additional host files and directories need to be
provided to the sensor by mounting them within the container:

```
/var/run/docker.sock    # for the sensor to query Docker engine
/var/log                # for logs
/etc/os-release         # or its equivalent based on the distro
```

Sensor container picks up `falconctl` configuration from its environment.
Following variables are supported.  They map to the similarly named
`SET_OPTIONS` of `falconctl`.  Typically, the environment variables
are set through a Kubernetes `configmap`.  They can also be set with
`-e` option to `docker run` on the command line.

```
FALCONCTL_OPT_CID
FALCONCTL_OPT_AID
FALCONCTL_OPT_APD
FALCONCTL_OPT_APH
FALCONCTL_OPT_APP
FALCONCTL_OPT_TRACE
FALCONCTL_OPT_FEATURE
FALCONCTL_OPT_MESSAGE_LOG
FALCONCTL_OPT_BILLING
FALCONCTL_OPT_ASSERT
FALCONCTL_OPT_MEMFAIL_GRACE_PERIOD
FALCONCTL_OPT_MEMFAIL_EVERY_N
```

The sensor can be run as a background service as follows:

```
CONTAINER_ID=$(docker run -d -e FALCONCTL_OPT_CID=<<your CID>> -e FALCONCTL_OPT_TRACE=debug --privileged --net=host -v /var/log:/var/log falcon-sensor:0.1)
```

### Running `falconctl`

`falconctl` can be invoked inside a running sensor container with `docker exec`:

```
docker exec -it $CONTAINER_ID falconctl -g --trace
```

## Post-reqs
Push the image to a registry (like ECR) if the container needs to be accessed outside of the build host.