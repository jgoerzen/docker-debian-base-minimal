# Debian Working System for Docker

**NOTE: Github is no longer the home for this project; see the [new home on Salsa](https://salsa.debian.org/jgoerzen/docker-debian-base)**.

This image is part of the
[docker-debian-base](https://salsa.debian.org/jgoerzen/docker-debian-base)
image set.

This is a simple set of images that transform the standard Docker
Debian environment into one that provides more traditional full
Unix APIs (including syslog, zombie process collection, etc.)

Despite this, they are all very small, both in terms of disk and RAM usage.

You can find a [description of the motivation for these images](https://changelog.complete.org/archives/9794-fixing-the-problems-with-docker-images) on my blog.

This is loosely based on the concepts, but not the code, in the
[phusion baseimage-docker](https://github.com/phusion/baseimage-docker).
You can look at that link for additional discussion on the motivations.

You can find the source and documentation at the [Salsa page](https://salsa.debian.org/jgoerzen/docker-debian-base)
and automatic builds are available from [my Docker hub page](https://hub.docker.com/u/jgoerzen/).  The builds are auto-generated from Salsa CI and run at least weekly.

**OUTDATED**: For stretch and jessie, these images used sysvinit instead of systemd.  If you are still using these extremely old images, consult the [old version of documentation](https://salsa.debian.org/jgoerzen/docker-debian-base/-/blob/ff9b50581f076f35c1be2b28828c98040ce57de0/README.md) for information about them.  Newer images are systemd exclusive and the information here will be incorrect for those very old ones.

The older images used sysvinit,
not because of any particular opinion on the merits of them, but
rather because sysvinit did not require any kind of privileged Docker
or cgroups access.

For newer releases, systemd contains the necessary support for running in an
unprivileged Docker container and, as it doesn't require the hacks
that sysvinit does, is used there.  The systemd and sysvinit images
provide an identical set of features and installed software, which
target the standard Linux API.

Here are the images I provide from this repository:

- [jgoerzen/debian-base-minimal](https://salsa.debian.org/jgoerzen/docker-debian-base-minimal) - a minimalistic base.
  - Provides working sysvinit/systemd, syslogd, cron, anacron, at, and logrotate.
  - syslogd is configured to output to the docker log system by default.
- [jgoerzen/debian-base-standard](https://salsa.debian.org/jgoerzen/docker-debian-base-standard) - adds some utilities.  Contains everything above, plus:
  - Utilities: less, nano, vim-tiny, man-db (for viewing manpages), net-tools, wget, curl, pwgen, zip, unzip
  - Email: exim4-daemon-light, mailx
  - Network: netcat-openbsd, socat, openssl, ssh, telnet (client)
- [jgoerzen/debian-base-security](https://salsa.debian.org/jgoerzen/docker-debian-base-security) - A great way to keep things updated.  Contains everything above, plus:
  - automated security patches using unattended-upgrades and needrestart
  - debian-security-support
  - At container initialization, runs the unattended-upgrade code path to ensure that the
    system is up-to-date before services are exposed to the Internet.  This addresses an
    issue wherein security patches may hit security.debian.org before Docker
    images are refreshed, a fairly common issue with the Docker infrastructure.
    This behavior can be suppressed with `DEBBASE_NO_STARTUP_APT` (see below).
- [jgoerzen/debian-base-vnc](https://salsa.debian.org/jgoerzen/docker-debian-base-vnc) - For systems that need X.  debian-base-security, plus:
  - tightvncserver, xfonts-base, lwm, xterm, xdotool, xvnc4viewer
- [jgoerzen/debian-base-apache](https://salsa.debian.org/jgoerzen/docker-debian-base-apache) - A web server - debian-base-security, plus:
  - apache2 plus utilities: ssl-cert
  - LetsEncrypt options: certbot, acme-tiny
- [jgoerzen/debian-base-apache-php](https://salsa.debian.org/jgoerzen/docker-debian-base-apache-php) - debian-base-apache, plus:
  - libapache2-mod-php (mod-php5 on jessie)
- [jgoerzen/debian-base-gemini](https://salsa.debian.org/jgoerzen/docker-debian-base-gemini) - debian-base-security, plus:
  - molly-brown, twins, gmnhg, md2gmn, md2gemini

Memory usage at boot (stretch):

- jgoerzen/debian-base-minimal: 6MB
- jgoerzen/debian-base-standard: 11MB
- jgoerzen/debian-base-security: 11MB

# Docker Tags

These tags are autobuilt:

 - latest: whatever is stable (currently bookworm, systemd)
 - bookworm: Debian bookworm (systemd)
 - bullseye: Debian bullseye (systemd)
 - buster: Debian buster (systemd) - **no longer supported, may be removed at any time**
 - stretch: Debian stretch (sysvinit) - **no longer supported, may be removed at any time**
 - jessie: Debian jessie (sysvinit) - **no longer supported, may be removed at any time**
 - sid: Debian sid (not tested; systemd)

# Install

You can install with:

    docker pull jgoerzen/debian-base-whatever

Your Dockerfile should use CMD to run `/usr/local/bin/boot-debian-base`.

When running, use `-t` to enable the logging to `docker logs`

# Container Invocation

A container should be started using these commands, among others.  See
also the section on environment variables, below.

## Container Invocation, systemd containers (buster/bullseye/bookworm/sid)

For a host running bullseye or bookworm, or a newer cgroups and systemd, you invoke like this:

    docker run -td --stop-signal=SIGRTMIN+3 \
      --tmpfs /run:size=100M --tmpfs /run/lock:size=100M \
      -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns=host \
      --name=name jgoerzen/debian-base-whatever

Here's how you invoke on a system running an older systemd on the host, with cgroups v1:

    docker run -td --stop-signal=SIGRTMIN+3 \
      --tmpfs /run:size=100M --tmpfs /run/lock:size=100M \
      -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
      --name=name jgoerzen/debian-base-whatever
      
bookworm is now the recommended image for all modern deployments.

The `/run` and `/run/lock` tmpfs are required by systemd.  The 100M
sets a maximum size, not a default allocation, and serves to limit the
amount of RAM an errant process could cause the system to consume,
down from a default limit of 16G.

Note that these images, contrary to many others out there, do NOT
require `--privileged`.

For more information about the systemd/cgroups situation, consult these links

- https://github.com/systemd/systemd/issues/19245
- https://github.com/containers/podman/issues/5153
- https://github.com/moby/moby/issues/42275
- https://serverfault.com/questions/1053187/systemd-fails-to-run-in-a-docker-container-when-using-cgroupv2-cgroupns-priva/1054414#1054414
- http://docs.podman.io/en/latest/markdown/podman-run.1.html#cgroupns-mode

# Environment Variables

This environment variable is available for your use:

 - `DEBBASE_SYSLOG` defaults to `stdout`, which redirects all syslog activity
   to the Docker infrastructure.  If you instead set it to `internal`, it will
   use the default Debian configuration of logging to `/var/log` within the
   container.  The configuration is applied at container start time by
   adjusting the `/etc/syslog.conf` symlink to point to either `syslog.conf.internal` or
   `syslog.conf.stdout`.  `syslog.conf.internal` is the default from the system.
   `dpkg-divert` is used to force all packages' attempts to write to `/etc/syslog.conf`
   to instead write to `/etc/syslog.conf.internal`.
- `DEBBASE_TIMEZONE`, if set, will configure the `/etc/timezone` and `/etc/localtime`
  files in the container to the appropriate timezone.  Set this to the desired timezone;
  for instance, `America/Denver`.
- `DEBBASE_SSH` defaults to `disabled`.  If you set to `enabled`, then the SSH server
  will be run.
- `DEBBASE_NO_STARTUP_APT` defaults to empty.  If set, it will cause images based
  on debian-base-security to skip the apt job run at container startup.

# Container initialization

Executables or scripts may be placed in `/usr/local/preinit`, which will be executed
at container start time by `run-parts` prior to starting init.  These can
therefore perform container startup steps.  A script which needs to only run
once can delete itself after a successful run to prevent a future execution.

# Orderly Shutdown

The `--stop-signal` clause in the "Container Invocation" section above
helps achieve an orderly shutdown.

If you start without `--stop-signal`, you can instead use these steps:

    docker kill -s SIGRTMIN+3 container

    # Then proceed with:
    sleep 10
    docker kill container

Within the container, you can call `poweroff` to cause the container to
shutdown.

## Advanted topic: Orderly Shutdown Mechanics

By default, `docker stop` sends the SIGTERM (and, later, SIGKILL)
signal to PID 1 (init) iniside a container.  Neither sysvinit nor
systemd act upon this signal in a useful way.  This will shut down a
container, but it will not give your shutdown scripts the chance to
run gracefully.  In many situations, this is fine, but it may not be
so in all.

A workaround is, howerver, readily available, without modifying init.  These
images are configured to perform a graceful shutdown upon receiving
`SIGRTMIN+3`.


With systemd, no special code for all this is needed;
systemd handles it internally with no fuss.

# Configuration

Although the standard and security images run the SMTP and SSH servers,
they do not expose these to the Internet by default.  Both require
site-specific configuration before they are actually useful.

Because the SMTP service is used inside containers, but the SSH service
generally is not, the SSH service is disabled by default.

## Enabling or Disabling Services

You can enable or disable services using commands like this:

    systemctl disable ssh
    systemctl enable ssh

(Note, that in the case of ssh, the environment variable will cause
commands like this to be executed automatically on each container
start.)

## Email

email is the main thing you'd need to configure.  In the running system,
`dpkg-reconfigure -plow exim4-config` will let you do this.

## SSH

SSH host keys will be generated upon first run of a container, if
they do not already exist.  This implies every instantiation
of a container containing SSH will have a new random host key.
If you want to override this, you can of course supply your own
files in `/etc/ssh` or make it a volume.

# Advanced Topic: Adding these enhancements to other images

Sometimes, it is desirable to not have to rebuild an image entirely.
These images are also designed to make it easy to add the
functionality to other images.  You can do this by using the support
for multiple FROM lines in a Dockerfile.  For instance, here's a
simple one I worked up:

    FROM jgoerzen/debian-base-security:jessie AS debian-addons
    
    FROM homeassistant/home-assistant:0.63.1

    COPY --from=debian-addons /usr/local/preinit/ /usr/local/preinit/
    COPY --from=debian-addons /usr/local/bin/ /usr/local/bin/
    COPY --from=debian-addons /usr/local/debian-base-setup/ /usr/local/debian-base-setup/
    
    RUN run-parts --exit-on-error --verbose /usr/local/debian-base-setup
    CMD ["/usr/local/bin/boot-debian-base"]

It happens that home-assistant is based on a Python image which, in
turn, is based on Debian jessie.  There are just those four lines that
are needed: copying the /usr/local/preinit, bin, and debian-base-setup
directories, and then the `run-parts` call.  This effectively adds all
the features of debian-base-security to the home-assistant image.

This works because each image that is part of the chain leading up to
security (minimal, standard, and security) performs all of its
activity from scripts it drops -- and leaves -- in
`/usr/local/debian-base-setup`.  Those scripts need nothing other than
the files in the three directories referenced above.  By adding those
three directories and calling the scripts, it is easy to add these
features to other images.

# Source

This is prepared by John Goerzen <jgoerzen@complete.org> and the source
can be found at https://salsa.debian.org/jgoerzen/docker-debian-base

# See Also

Some references to additional information:

 - systemd's
   [contianer interface documentation](https://www.freedesktop.org/wiki/Software/systemd/ContainerInterface/)
 - [Article](https://developers.redhat.com/blog/2016/09/13/running-systemd-in-a-non-privileged-container/)
   on running systemd in a container.  Highlights some of the reasons
   to do so: providing a standard Linux API, reaping zombie processes,
   handling of logging, not having to re-implement init, etc.  All of
   these have already been implemented in these images with sysvinit
   and continue with systemd.
 - [serverfault thread](https://serverfault.com/questions/607769/running-systemd-inside-a-docker-container-arch-linux)

# Copyright

Docker scripts, etc. are
Copyright (c) 2017-2024 John Goerzen
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:
1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.
3. Neither the name of the University nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE AUTHORS AND CONTRIBUTORS ``AS IS'' AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
SUCH DAMAGE.

Additional software copyrights as noted.

