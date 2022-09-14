# YaST Management Containers

<!--
TODO: later move this documentation to the containers Git repository,
that's related to https://trello.com/c/AVE56dla
-->

*Note: Originally this repository contained a helper script for starting the
YaST containers. But that script was replaced by the RUN container label.
If you want to see the original content check the [*master_old*](
../../tree/master_old) branch.*


## :warning: Warning :warning:

**This project is still in development, do not use it in production systems!
It is recommended to use a virtual machine for testing! There is a risk of
breaking the system or data loss!**

## Purpose

The goal of this project is to decrease the size of the system and to avoid
unnecessary dependencies.

Using a separate container would also make upgrading the tools, libraries
and languages easier, possibly without affecting the users. We could use a newer
Ruby in the YaST container and keep the old one in SLE/Leap for backward
compatibility.

## List of Container Images

There are several images available, depending which user interface you would
like to use:

- `yast-mgmt-ncurses`: the base container image, contains YaST with text mode
  interface (ncurses)
- `yast-mgmt-qt`: this image adds the graphical user interface (Qt-based)
- `yast-mgmt-web`: adds web access by exposing the standard graphical interface
  in a VNC server and using Javascript VNC client to render the screen in
  a browser

### Testing Containers

These images are intended for testing the containers themselves.

- `yast-mgmt-ncurses-test-api`: adds the REST API used to control the text-based
  interface
- `yast-mgmt-qt-test-api`: equivalent to the previous one but with the graphical
  Qt interface

These containers should be used by openQA or other automated tools. They should
not be used in production systems because using the remote controlling API has
some security consequences.

## Pre-requisites

A container runtime is required to run YaST in a container. Both [Podman](
https://podman.io/) and [Docker](https://www.docker.com/) are supported.

However, using Podman is recommended as it can start the containers in an easier
way.

### Installing Podman

```shell
# install the package
zypper install podman
```

### Installing Docker

```shell
# install the package
zypper install docker
# enable and start the Docker service
systemctl enable docker
systemctl start docker
```

If you want to use Docker as a non-root user you can add yourselves into the `docker`
user group. *(Security note: Be careful, such users become equivalent to `root`!)*


## Starting the Containers

Here are the commands for starting the YaST container. If you want to use the
other images than just replace `ncurses` with `qt` or `web`.

### Podman

Running in Podman is easy:

```shell
podman container runlabel run registry.opensuse.org/suse/alp/workloads/tumbleweed_containerfiles/suse/alp/workloads/yast-mgmt-ncurses:latest
```

If you want to see the command which will be executed in advance then use the
`--display` option:

```shell
podman container runlabel --display run registry.opensuse.org/suse/alp/workloads/tumbleweed_containerfiles/suse/alp/workloads/yast-mgmt-ncurses:latest
```

### Docker

Docker does not support using the `RUN` label directly. We need to manually
extract it, modify it for Docker and then start:

```shell
# the image to use
IMAGE="registry.opensuse.org/suse/alp/workloads/tumbleweed_containerfiles/suse/alp/workloads/yast-mgmt-ncurses:latest"
# download the image
docker pull $IMAGE
# get the RUN label from the image, replace "podman" with "docker",
# replace "IMAGE" with the container image name and remove the "--replace"
# option which is not supported by Docker
DOCKER_COMMAND=`docker inspect --format="{{.Config.Labels.RUN}}" $IMAGE | sed -e s/podman/docker/ -e s@IMAGE@$IMAGE@ -e s/--replace//`
# print the command
echo $DOCKER_COMMAND
# if it is OK then run it
$DOCKER_COMMAND
```

## Remote Access

The `yast-mgmt-web` container sets up a web access to the running YaST.
It runs the YaST using the graphical UI but the UI is accessed via a web
interface instead of the local X server.

That obviously means the web browser will display the usual Qt UI, there will
be no special web UI with modern features like responsive design or similar.

### Setting the Access Password

<!-- TODO: this needs to be updated, see related https://trello.com/c/vpVSyV4f -->

The access is password protected, there are several ways how the set the password.

If you are using podman then you can setup secrets like this:

- `printf "<password>" | podman secret create yast_vnc_password -` \
  This stores the password in plain text
- `/usr/bin/vncpasswd.arg /dev/stdout "<password>" | podman secret create yast_vnc_password_bin -` \
  This stores the password encrypted, this should be more secure than a plain
  text password

Another possibility, which works also with Docker, is to store the password in
the host

```shell
/usr/bin/vncpasswd.arg /root/.vnc/passwd.yast "<password>"
```

:information_source: *Note: The `vncpasswd.arg` tool is included in the
`xorg-x11-Xvnc` package.*

:warning: *Warning: Use a strong password, YaST runs with administrator
permissions and everybody who can access the web interface has full
administrator access to the machine!*

### Setting the SSL Certificate

The web server uses the secure HTTPS connection. This ensures that the
connection between the browser and the server is encrypted and nobody can read
the communication.

This requires an SSL certificate. You can provide your own valid certificate,
for example using free [Let's Encrypt](https://letsencrypt.org/) certificate
authority. If you do not specify your certificate then the starting script will
generate a self-signed certificate. The problem is that the browsers will
complain about unknown certificate authority.

:warning: *When using a self-signed certificate you should compare the
certificate fingerprint printed by the starting script with the certificate
fingerprint reported by the browser. If they do not match then do not use
that connection!*

You need certificate in a combined PEM format which contains both the
certificate and the secret key. If you are using podman then you can setup
the secret like this:

```shell
podman secret create yast_vnc_ssl_cert ./certificate.pem
```

Another option, which also works with Docker, is to save the certificate
to the `/root/.vnc/cert.pem` file in the host system.


## Supported Modules

Not all YaST modules can be directly used from a container, usually they need
some small adjustments to work properly.

The YaST control center in the containers displays only the supported modules.
Alternatively you can run `yast2 -l` command in the containers to list the
supported modules.

The container image might contain some more modules which are not displayed
in the control center. These modules have been installed by dependencies but
they are not supported and will probably not work properly, do not use them.

## Debugging

If you need to run a specific command in the container instead of the YaST
control center or an YaST module then you need to disable the entry point script
and pass your command on the command line.

- Get the command for starting the container (see the examples above)
- Add the `--entrypoint=""` option after the `run` argument
- Add the command to run at the end, e.g. `bash`


## Implementation Details

<!--
TODO: move this to a separate document for developers, this should be mainly the
user's documentation.
-->

### Accessing the Host System

The host system is mounted to the `/host` directory in the container. YaST must use
this subdirectory instead of the root directory for reading/writing files.

But that's usually not supported, esp. for modules designed to run only in
an installed system.

### Package Management

Libzypp supports installing into a chroot directory, that's used during standard
installation. That means the package management should work fine.

But the problem is that YaST needs to distinguish between the packages needed
in the management container and the packages needed in the host system.

The partitioning tools could be used from the container, but on the other hand,
configuring NetworkManager requires it in the host system so it can set the
network after reboot.

Unfortunately this distinction is missing in YaST...

### Executing commands

Similarly to the package management, it is important to know where the executed
command is available and where it needs to run.

- Run the command in the container (e.g. `fdisk` can be used from a container
  and does not need access to the host files)
- Run it in the container and pass it `/host` option if that is possible so
  it modifies the host system
- Run it in the container but copy the result into the target system
- Run it in the host system using `chroot` (but that requires the command to be
  available in the host system)
- Run the command using SSH, this would ensure it is fully executed in the host
  system context, but that would require SSH to be installed and running.

### Other Interactions with the System

It is a question which other interactions with the host system are possible
or not. It turned out that even loading kernel modules works with
`chroot /host modprobe <module>`...

### Logging and Locking

The YaST log is written into `/var/log/YaST2/y2log` in the host as usually.

The libzypp lock (`/var/run/zypp.pid`) is also created in the hosts system,
this avoids running multiple instances of the package management at once.

### Implementation

The implementation uses these two important components:

- [Xvnc](https://tigervnc.org/doc/Xvnc.html) - an X server which creates virtual
screens which can be accessed via the VNC protocol
- [novnc](https://novnc.com/info.html) - a VNC client written in pure JavaScript
which can run in a browser, it is connected to the Xvnc server

The connection between these two components is realized using an Unix socket file,
the VNC service cannot be accessed from outside and is completely hidden inside
the container to make it more secure.

## Links

- https://github.com/yast/yast-ruby-bindings/pull/284 - automatic chroot support
  in the YaST starting script
- https://github.com/yast/yast-yast2/pull/1258 - automatically set the target
  system location when chrooted
- https://github.com/yast/yast-registration/pull/577 - adapt the registration
  module to run in a chroot
- https://github.com/yast/yast-packager/pull/621 - adapt the repository and package
  management modules to run in a chroot
- https://build.opensuse.org/package/show/YaST:Head/yast-mgmt-ncurses-leap_latest - this
  OBS project builds the base container image with ncurses UI
- https://build.opensuse.org/package/show/YaST:Head/yast-mgmt-qt-leap_latest - this
  OBS project extends the ncurses container image with Qt and X libraries so it can
  run the graphical UI
- https://build.opensuse.org/package/show/YaST:Head/yast-mgmt-web-leap_latest - this
  OBS project adds the `Xvnc` and `novnc` packages and an initialization script
  which starts X session accessible using a web browser
