# Debian Working System for Docker

This is a simple set of images that transform the standard Docker
Debian environment into one that provides more traditional full
Unix APIs (including syslog, zombie process collection, etc.)

Despite this, they are all very small, both in terms of disk and RAM usage.

You can find a [description of the motivation for these images](http://changelog.complete.org/archives/9794-fixing-the-problems-with-docker-images) on my blog.

This is loosely based on the concepts, but not the code, in the
[phusion baseimage-docker](https://github.com/phusion/baseimage-docker).
You can look at that link for additional discussion on the motivations.

You can find the source and documentation at the [Github page](https://github.com/jgoerzen/docker-debian-base)
and automatic builds are available from [my Docker hub page](https://hub.docker.com/u/jgoerzen/).

This image uses sysvinit instead of systemd, not because of any
particular opinion on the merits of them, but rather because
sysvinit does not require any kind of privileged Docker
or cgroups access.  

Here are the images I provide from this repository:

- jgoerzen/debian-base-minimal - a minimalistic base for you.
  - Provides working sysvinit, syslogd, cron, anacron, at, and logrotate.
  - syslogd is configured to output to the docker log system by default.
- jgoerzen/debian-base-standard - adds some utilities.  Containes everything above, plus:
  - Utilities: less, nano, vim-tiny, man-db (for viewing manpages), net-tools, wget, curl, pwgen, zip, unzip
  - Email: exim4-daemon-light, mailx
  - Network: netcat-openbsd, socat, openssl, ssh, telnet (client)
- jgoerzen/debian-base-security - A great way to keep things updated.  Contains everything above, plus:
  - automated security patches using unattended-upgrades and needrestart
  - debian-security-support
- jgoerzen/debian-base-vnc - For systems that need X.  debian-base-security, plus:
  - tightvncserver, xfonts-base, lwm, xterm, xdotool, xvnc4viewer
- jgoerzen/debian-base-apache - A web server - debian-base-security, plus:
  - apache2 plus utilities: ssl-cert
  - LetsEncrypt options: certbot, acme-tiny
- jgoerzen/debian-base-apache-php - debian-base-apache, plus:
  - libapache2-mod-php (mod-php5 on jessie)

Memory usage at boot (stretch):

- jgoerzen/debian-base-minimal: 6MB
- jgoerzen/debian-base-standard: 11MB
- jgoerzen/debian-base-security: 11MB

These images are autobuilt for jessie, stretch, and sid.

# Install

You can install with:

    docker pull jgoerzen/debian-base-whatever

Your Dockerfile should use CMD to run `/usr/local/bin/boot-debian-base`.

When running, use `-t` to enable the logging to `docker logs`

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
  files in the container to the appropriate timezone.
- `DEBBASE_SSH` defaults to `disabled`.  If you set to `enabled`, then the SSH server
  will be run.

# Container initialization

Executables or scripts may be placed in `/usr/local/preinit`, which will be executed
at container start time by `run-parts` prior to starting init.  These can
therefore perform container startup steps.  A script which needs to only run
once can delete itself after a successful run to prevent a future execution.

# Orderly Shutdown

You can cause `docker stop` to invoke an orderly shutdown by running the container
like this:

    docker run -td --stop-signal=SIGPWR --name=name jgoerzen/debian-base-whatever

If you don't start it this way, you can instead use these steps:

    docker kill -s SIGPWR container
    sleep 10
    docker kill container

Within the container, you can call `telinit 1` to cause the container to shutdown.

## Advanted topic: Orderly Shutdown Mechanics

By default, `docker stop` sends the SIGTERM (and, later, SIGKILL) signal to PID
1 (init) iniside a container.  sysvinit does not act upon this signal.
This will shut down a container, but it will not give your shutdown scripts
the chance to run gracefully.  In many situations, this is fine, but it may
not be so in all.

A workaround is, howerver, readily available, without modifying init.  These
images are configured to perform a graceful shutdown upon receiving `SIGPWR`.

The process for this is... interesting, since we are unable to directly
kill PID 1 inside a docker container.  First, init calls `/etc/init.d/powerfail`.
The powerfail script I install simply tells init to go to single-user mode.
This causes it to perform an orderly shutdown of the daemons, and when it is
done, it invokes `/sbin/sulogin`.  On an ordinary system, this prompts for
the root password for single-user mode.  In this environment, we instead
symlink /sbin/init to /bin/true, then tell init to re-exec itself.  This
causes PID 1 to finally exit.

One of the preinit scripts makes sure that `/sbin/init` properly links to
`/sbin/init.real` at boot time.

# Configuration

Although the standard and security images run the SMTP and SSH servers,
they do not expose these to the Internet by default.  Both require
site-specific configuration before they are actually useful.

Because the SMTP service is used inside containers, but the SSH service
generally is not, the SSH service is disabled by default.

## Enabling or Disabling Services

You can enable or disable services using commands like this:

   update-rc.d ssh disable 
   update-rc.d ssh enable

## Email

email is the main thing you'd need to configure.  In the running system,
`dpkg-reconfigure -plow exim4-config` will let you do this.

## SSH

SSH host keys will be generated upon first run of a container, if
they do not already exist.  This implies every instantiation
of a container containing SSH will have a new random host key.
If you want to override this, you can of course supply your own
files in /etc/ssh or make it a volume.

# Advanced topic: programs that depend on disabled scripts

There are a number of scripts in `/etc/init.d` that are normally
part of a Debian system initialization, but fail in a Docker environment.
They do things like set up swap space, mount filesystems, etc.
Docker images typically leave those scripts in place, but they are
never called because Docker systems typically don't run a real init
like these images do.

Although calling the scripts produces nothing worse than harmless errors, I have
disabled those scripts in these images in order to avoid putting useless
error messages in people's log files.  In some very rare circumstances,
this may cause installation of additional packages to fail due to
boot script dependency ordering not working right.  (Again, this is very
rare).

I saw this happen once where a package had a long chain of dependencies
that wound up pulling in cgmanager, which died in postinst complaining
that its init script required `mountkernfs`.  I worked around this in my
Dockerfile like this:

    update-rc.d mountkernfs.sh defaults
    apt-get -y --no-install-recommends offending-package
    update-rc.d -f cgmanager remove
    update-rc.d -f mountkernfs.sh remove

Also, I have blocked systemd from accidentally being installed on the system.
There are a few packages that pull in systemd shims and so forth, so if
you get errors about systemd not installing, try adding
`rm /etc/apt/preferences.d/systemd` to your Dockerfile.

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
/usr/local/debian-base-setup.  Those scripts need nothing other than
the files in the three directories referenced above.  By adding those
three directories and calling the scripts, it is easy to add these
features to other images.

# Docker Tags

These tags are pushed:

 - latest, stretch: Debian stretch
 - jessie: Debian jessie
 - sid: Debian sid (not tested)

# Source

This is prepared by John Goerzen <jgoerzen@complete.org> and the source
can be found at https://github.com/jgoerzen/docker-debian-base

# Copyright

Docker scripts, etc. are
Copyright (c) 2017-2018 John Goerzen
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

