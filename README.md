# How I start

This is a simple, hello world, python webapp, hosted in a tiny container based on Arch Linux. I use Arch as the host too,
but any other distro with systemd-nspawn should work. That includes Debian 8.x, Fedora, RHEL/CentOS 7.x, Ubuntu 15.04,
OpenSuse, etc…

## Why?

Just to show a possibly better alternative to Docker.

## HOW & WHAT

…

# Let's go

``` shell
MYAPP=$HOME/work/uwsgi-hello-world
CONTAINER=/tmp/container

git clone https://github.com/gdamjan/hello-world-python-container.git $MYAPP

cd $MYAPP
export PYTHONUSERBASE=$MYAPP/py-env
pip2 install --ignore-installed --user -r requirements.txt

# install pacstrap on the host
sudo pacman -S arch-install-scripts

# create the container. pacstrap will install the minimum needed for uwsgi python
mkdir -p $CONTAINER
sudo pacstrap -c -d -G $CONTAINER uwsgi uwsgi-plugin-python2
sudo rm -rf $CONTAINER/usr/share  # free some 90MB

# when $CONTAINER is on btrfs could use --ephemeral or --template
sudo systemd-nspawn \
    --machine myapp \
    --user nobody \
    --bind $MYAPP:/srv/app \
    -D $CONTAINER \
    /usr/bin/uwsgi --ini /srv/app/uwsgi.ini
```

To take a look from the perspective of the container use the `nsenter` utility. Here $PID is the
process id of the master uwsgi process as seen from the host side (it's pid 1 in the container).
Use `ps axf` to find it.

``` shell
sudo nsenter --target $PID --mount --uts --pid --ipc --net

# but perhaps for debugging you'll need some additional packages installed:
sudo pacman --root $CONTAINER -S util-linux procps-ng coreutils iproute2
```


# Future

* At one point systemd should implement descriptive unit file type for containers (similar to .service). This would make
  things even better and avoid running containers with the long systemd-nspawn command line.

* systemd-nspawn should support the --template and --ephemeral features with overlayfs in addition to btrfs. That should
  make things easier for people not using btrfs.

* When (if?) user namespaces kernel feature becomes more widely supported, even the use of `sudo` might not be needed.


# FAQ

#### Can I use virtualenv?

No, virtualenv is not relocatable, it woud be problematic to use it across containers and the host.
the [pep-0370](https://www.python.org/dev/peps/pep-0370/), PYTHONUSERBASE env var way is actually
simpler, standardized and much better supported. Works each time.

#### Updating the system?

`pacman --root $CONTAINER -Syu` - it's the host sys-admin responsibility.

#### PHP?

Sure, just install `uwsgi-plugin-php`. PHP apps don't need PYTHONUSERBASE of course, and are ussually self contained.
You don't even need the `open_basedir` restriction since there's nothing to see in the container (and it does have
a bad performance impact).

#### Ruby/Rack

Install `uwsgi-plugin-rack`. Use bundler to install things in `$MYAPP/vendor` - this seems to be a widely used Ruby
convention.

#### Debian, Ubuntu containers?

Yep, just use `debootstrap` as documented in the [systemd-nspawn](http://www.freedesktop.org/software/systemd/man/systemd-nspawn.html) man page. They do have older uwsgi packages though (maybe more stable?).

#### Debian, Ubuntu, RHEL, Fedora as hosts?

Haven't tried them. I'd assume they'll just work until further.


# Why not use Docker

I don't like Docker that much. It is a comfortable tool that makes it easy to start with it, but I see several
shortcommings for serious use. Docker seems to be 3 things (for now).

* A build tool - RUN commands in the Docker file are a really crude, suboptimal way to build images. Better tools are
needed or even available. A shell script with chroot, or a Makefile, or ansible can express much better how you build
things.

* A runtime - I have no trust in the Docker runtime, at all. Just take a look at what all can systemd do with
processes it manages. An indexed database of logs (journal), process control to the slightest detail. Docker
seems to me as a very mediocre process management system.

* The binary image hub - that's just awful, getting random binary blobs from random people on the internet.
Binary blobs running with basically root privileges. Also images on the hub are not updated (heartbleed anyone), and
often contain left-over unneeded files in them. The worst part of it is the culture it creates :(

PS. seems that Docker community will fix these issues in the future. There's also a lot of work and community around
the [Open Container](https://www.opencontainers.org/) standards.
