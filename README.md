# Experimental YaST Management Container

[![CI](https://github.com/yast/yast-in-container/actions/workflows/ci.yml/badge.svg)](
https://github.com/yast/yast-in-container/actions/workflows/ci.yml)

This is a proof of concept (PoC) project for testing how to run YaST in
a container to manage the host system.

## :warning: Warning :warning:

**This is an experimental project, do not use it in production systems! It is
recommended to use a virtual machine for testing! There is a high risk of
breaking the system or data loss!**

## Purpose

The goal of this project is to decrease the size of the system and to avoid
unnecessary dependencies.

Using a separate container would also make upgrading the tools, libraries
and languages easier, possibly without affecting the users. We could use a newer
Ruby in the YaST container and keep the old one in SLE/Leap for backward
compatibility.

## Proof of Concept

*Note: The built containers use the openSUSE Leap 15.4 system and should be used
only for managing openSUSE Leap 15.4 or SLE-15-SP4 systems!*

You can disable this product check by setting `VERSION_CHECK=0` environment
variable. Be very carful when doing this, use it on your own risk!!

## Pre-requisites

A container runtime is required to run YaST in a container. Both [Podman](
https://podman.io/) and [Docker](https://www.docker.com/) are supported.

### Podman

Note: Podman is automatically recommended by the `yast-in-container` package.

```shell
# install the package
zypper install podman
```

### Docker

Alternatively the scripts can use [Docker](https://www.docker.com/) for
managing and running containers.

```shell
# install the package
zypper install docker
# enable and start the Docker service
systemctl enable docker
systemctl start docker
```

If you want to use Docker as a non-root user you can add yourselves into the `docker`
user group. *(Security note: Be careful, such users become equivalent to `root`!)*

## Installing the Scripts

The scripts can be installed either from YaST OBS YaST:Head repository or can be
started directly from the Git sources.

Run as `root`:

```sh
# add the YaST:Head repository
zypper addrepo -f 'https://download.opensuse.org/repositories/YaST:/Head/openSUSE_Leap_${releasever}/' YaST:Head
# install the package
zypper install yast-in-container
# disable the repository to not accidentally install some other YaST packages
zypper modifyrepo -d YaST:Head
# run the script (from /usr/sbin/ or /sbin)
yast2_container repositories
```

```sh
# download the scripts
git clone https://github.com/yast/yast-in-container.git
cd yast-in-container/src/scripts
# run the script
./yast2_container repositories
```

## Details

This repository provides several  helper scripts

- `yast_container` - runs YaST with the ncurses (text mode) UI
- `yast2_container` - runs YaST with the Qt (graphical mode) UI
- `yast2_web_container` - runs YaST inside Web session, open the
  https://localhost:4984 URL in a web browser, see more
  [details below](#web-https-access)

The scripts download the container image from OBS and then run the specified
YaST module in the container. If no YaST module is specified then the YaST
control center is started.

The container is automatically deleted after finishing YaST, if you want to
inspect the container you have to start it manually.

## Debugging

If you need to run a different command than YaST in the container for testing
purposes you can specify it with the `CMD` environment variable.

```sh
# run shell for an interactive session instead of running YaST
CMD=bash yast2_container
```

The script prints the executed `podman` or `docker` command, if you need to
run the container with different options just copy&paste the printed command
to the terminal.

## Tested Modules

This is a list of YaST modules which you can try with the testing image:

- `yast2_container scc` - can register the system against SCC, RMT/SMT are also
  supported, importing a self-signed SSL certificate works properly
- `yast2_container repositories` - shows and edits the repositories from
  the host system, should fully work
- `yast2_container sw_single` - package installation works fine,
  tha packages are correctly installed in the host system
- `yast2_container key_manager` - displays the imported GPG keys
  known by the package management

## Implementation Details

### Accessing the Host System

The host system is mounted to `/mnt` directory in the container. YaST must use
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
- Run it in the container and pass it `/mnt` option if that is possible so
  it modifies the host system
- Run it in the container but copy the result into the target system
- Run it in the host system using `chroot` (but that requires the command to be
  available in the host system)
- Run the command using SSH, this would ensure it is fully executed in the host
  system context, but that would require SSH to be installed and running.

### Other Interactions with the System

It is a question which other interactions with the host system are possible
or not. It turned out that even loading kernel modules works with
`chroot /mnt modprobe <module>`...

### Logging and Locking

The YaST log is written into `/var/log/YaST2/y2log` in the host as usually.

The libzypp lock (`/var/run/zypp.pid`) is also created in the hosts system,
this avoids running multiple instances of the package management at once.

## Web (HTTPS) Access

The `yast2_web_container` script sets up a web access to the running YaST.
It runs the YaST using the graphical UI but the UI is accessed via a web
interface instead of the local X server.

That obviously means the web browser will display the usual Qt UI, there will
be no special web UI with modern features like responsive design or similar.

### Setting the Access Password

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

### Implementation

The implementation uses these two important components:

- [Xvnc](https://tigervnc.org/doc/Xvnc.html) - an X server which creates virtual
screens which can be accessed via the VNC protocol
- [novnc](https://novnc.com/info.html) - a VNC client written in pure JavaScript
which can run in a browser, it is connected to the Xvnc server

The connection between these two components is realized using an Unix socket file,
the VNC service cannot be accessed from outside and is completely hidden inside
the container to make it more secure.

## Summary

It seems that it should be possible to fully manage the host system from
a container.

However, the current YaST is not fully prepared for that. The adjustments seem to
be small but a lot of places would need to be updated. It should be similar
to moving some configuration steps from the second installation stage
(running in `/`) to the first stage (running in `/mnt`) as we did some time ago.

Also some YaST modules do not make sense in a containerized world,
for example the module for HTTP server. It is supposed that the HTTP server
would run in a separate container, possibly managed by other tools.

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
