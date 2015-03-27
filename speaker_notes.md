# Speaker notes for CDATX Advanced Training.

[http://tech.paulcz.net/cdatx-advanced-docker](http://tech.paulcz.net/cdatx-advanced-docker)

## 1 - Welcome

## 2 - Who am I?

## 3 - BBC

## 4 - Agenda

## 5 - Intro - Optimizing dockerfiles

## 6 - Intro

Docker images are “supposed” to be small and fast. However unless you’re precompiling GO binaries and dropping them in the busybox image they can get quite large and complicated. Without a well constructed Dockerfile to improve build cache hits your docker builds can become unnecessarily slow.

Dockerfile’s are regularly [and incorrectly] treated like bash scripts and therefore are often written out as a series of commands which you would curl | sudo bash from a website to install. This usually makes for an inefficient and slow Dockerfile

## 7 - Example dockerfile

Make sure everybody understands what all is going on.   Ask someone to talk through it.

## 8 - Order Matters

When you’re building a new Dockerfile for an application there can be a lot of trial and error in determining what packages are needed and what commands need to run. Optimizing your Dockerfile ensures that the build cache will hit more often and each build between changes will be faster.

## 9 - Order Matters

The general rule of thumb is to sort your commands by frequency of change, the time it takes to run the command and how sharable it is with other images.

## 10 - Order Matters

Sny ADD ( or other commands that invalidate cache ) commands should go as far down the bottom as possible as this is where you’re likely to make lots of changes that will invalidate the cache of subsequent commands.

Commands like WORKDIR, CMD, ENV should go towards the bottom while a RUN apt-get -y update should go towards the top as it takes longer to run and can be shared with all of your images.


## 11 - Choose base images wisely

There’s a lot of base images to choose from from the bare OS images like ubuntu:trusty to application specific ones for python:2 or java:7. Common sense might tell you to use ruby:2 to run an ruby based app and python:3 to run a python app. However now you have two base images with little in common that you need to download and build. Instead if you use ubuntu:trusty for both then you only need to download the base image once.

## 12 - Use Layers to your advantage

Each command in a Dockerfile is an extra layer. You can very quickly end up with an image that’s 30+ layers. This is not necessarily a problem, but by joining RUN commands together, and using a single EXPOSE line to list all of your open ports you can reduce the number of layers.

By grouping RUN commands together intelligently you can share more layers between containers. Of course if you have a common set of packages across multiple containers then you should look at creating a seperate base image containing these that all of your images are built from.

For each layer that you can share across multiple images you can save a ton of disk space.

## 13 - Cheat

If you’ve built an image and discover when you run it that there’s a package missing add it to the bottom of your Dockerfile rather than in the RUN apt-get command at the top. This means you can rebuild the image faster. Once your image is correct and working you can reorganize your Dockerfile to clean such changes up before commiting it to source control.

## 14 - Volume containers

If you use Volume containers, don’t bother trying to save space by using a small image, Use the image of the application you’ll be serving data to. If you do that and docker commit the data volume you not only have your data commited to the container, but the actual application as well which is very useful for debugging.

## 15 - pop quiz slide

## 16 - example dockerfile

Ask class to re-order this Dockerfile.

## 17 - Agenda Docker vs Config Management

## 18 - I don't need...

talk to fallacy of this statement.

* manage non docker
* greenfield vs brownfield
* Skeuomorph

## 19

## 20 - Persistent Data

Docker is not [yet] good place for data to live

* docker containers are ephemeral/immutable by design
* AUFS didn't write when it say it did
* docker commit to ensure data is written to disk

* bind mounting from host came about.  Gabe @ Deis
    * until docker supports user name spaces this data owned by root

* volume mounts from other containers
    * only useful when on same host
    * all same issues as data being in container
        * except it persists after container stops.

## 21 - Brownfield / infrastructure

* Persistent data and other infrastructure services have to be managed by something.  
* If not docker, then VMs or baremetal
* now we need orchestration / CM
    - Configure and run applications - mysql, haproxy
    - Configure and deploy services - RDS, ELB

## 22 - docker cookbook

* really strong cookbook that installs docker
* has a bunch of LWRPs to allow you to perform most ( if not all) docker commands from within chef.
* Most people I know that are running non-trivial systems that include docker are using this. -  See Drama Fever

## 23 - docker cookbook example

Example registry recipe

* pull the docker registry image
* run a docker registry container 
    - if backed by object store, run on every node
* pull a list of images from the public docker registry
* push them to the private registry

## 24 - docker cookbook example

Example 'trusted build' recipe

* clones a git repo containing a Dockerfile
* when that changes, it notifies docker build to build an image
* could easily extend this to push the resultant image to registry - public or private.

## 25 - anything you can do i can do better

Almost anything that can be done with docker can be done via the docker cookbook.

Puppet and Ansible both have a similar story, but I haven't used them.

## 26 - CM in the container

You probably shouldn't.   But if you need to, or it makes sense to from where you are ... then ignore the haters and do it.

* Chef for Containers was a thing
* Helped provision inside the container and provided a lightweight init system.
* recently ( like last week ) it was depreciated as a standalone thing and is being wrapped into chef-provisioning.  I don't know how/if the functionanility will change.

* Bind mount chef-client into the container
* Have chef client locally or in a volume container
* bind mount it in to run chef-client in a base linux container
* commit the resulting container and push it to registry. 

## 27 - CM to build containers

* if you already subscribe to the immutable infrastructure and burn AMIs or simply burn AMIs to reduce CM converge time you're probably already using a tool like packer.
* packer supports docker and you can probably, with very little tweaking tell it to build packer instead.

## 28 - ChefDK in a container

ChefDK is still fairly young and support for it is really focussed around OSX.  There's some Linux support, but it's for outdated, even no longer supported versions of Ubuntu ...  If you have anything else ... GOOD LUCK TO YOU SIR.

One way to work around this is to run chefdk in a ubuntu 12.04 container and bind mount in your current cookbook directory.   now you can hit it with `docker exec -ti chefdk rake style spec`.   Some things might not work easily like virtualbox support for testkitchen,  but most things do work.

## 29 - agenda screen

## 30 - Logging and Metrics

In the early days of docker both logging and metrics gathering was very difficult.  there was very little support in docker and no real tools in the ecosystem for what did exist.

Thankfully this has changed and Docker has some good ( and getting better ) support for both logging and metrics.

## 31 - Logging

The first thing you want to be really cognizant about is that you should do your very best to never actually log to a file in a container in the first place.  

Otherwise A long living container will start to need log management, rotation, space management, etc to make sure that you keep it under control

## 32 - Logspout

Logspout was created by Jeff Lindsay who has had his hand in just about every awesome thing in the Docker ecosystem since before docker was even an actual thing.

* watches the docker APIs log stream
* runs in a container - bind mount in the docker socket
* forwards logs to one or more syslog servers defined at runtime

## 33 - /dev/stdout

If you app insists on logging to a file you can set it to  use /dev/stdout and /dev/stderr which do exactly what you might guess, any text written to them with through the magic of linux show up in the process' stdout/stderr.

## 34 - but even then I can't

I discovered once that mysql insists on logging errors to a file with extension of .err if you didn't specify an extension.  I had to symlink /dev/stderr to /var/log/mysql.err.

You can also run syslog in the container itself as a second process and have your applications log to syslog instead.   haproxy has a hard time logging to anything that's not syslog,  this can help with that.

There's an issue filed against logspout for creating a lightweight syslog server that such containers can use as a syslog target and thus route through logspout the same as other containers.

worse case scenario you can always mount a volume from the host or a volume container and log to there.   It's not ideal but it moves responsibility for managing log files to outside the container running the app.

## 35 - logging improvements coming soon

1.6 should have improved logging,  will support logging drivers which should allow for native transport via syslog and other methods.

## 36 - Metrics

Until very recently ( 1.5 ) docker didn't have any native support for collecting metrics.   However because docker is a wrapper around LXC we have access to all sorts of metrics if we know where to you.

LXC exposes metrics in `/proc` and `/sys` and other weird places that you probably don't know to look.   Example memory stats for a given container would go to `/sys/fs/cgroup/memory/lxc/[container_id]/`.

As I said,  you really have to know LXC well to know where all the stats end up.

## 37 - Metrics - CAdvisor

Google had already solved the metrics problem internally as they are one of the early pioneers of LXC and contributed a lot of the kernel code for cgroups etc.

As you can see from the docker command here its pretty intrusive, needing a lot of volumes mounted into it,  thankfully most can be mounted read only.

## 38 - Metrics - Cadvisor

Cadvisor collects metrics for any running containers ( and not just docker containers ) and provides access to them via API or GUI and can push them to a couple of different metrics collection engines, currently InfluxDB and Prometheus,   I would expect to see support for graphite at some point as well.

It collects per container metrics for the usual suspects of interesting machine properties such as CPU, Memory, Network, Disk.

## 39 - agenda slide

## 40 - multiprocess containers

So this is a highly charged topic.  You here a lot of noise telling you to only ever run a single process in a container, and if you run more than that you're doing it wrong.  I would go as far as to say the rhetoric for this argument gets quite poisonous.     

Docker themselves have said countless times that they see room for both single process containers and multi-process containers.   I am firmly on this side of the bench.   

I will agree that the fewer processes the better, and if you happen to be greenfielding you should really be paying attention to the whole 12factor mindset,  but most of us are in situations where this is not entirely possible.

There's a few ways to achieve this, and they each have varying levels of merit and issues which make them attrative for different reasons.

## 41 - backgrounded processes

The easiest way to run multiple containers is to have a short shell script that you run inside the container.  This can be added in your dockerfile and set to ENTRYPOINT or CMD.

It simply runs the first command with an & and then the second command in the foreground like so.

## 42 - problems with backgrounded processes

There's a few problems with doing this, so I wouldn't recommend it for anything other than a quick dev or test container.  If its not going to be long lived, then this is by far the easiest way to achieve multiple processes.

Problems:

* Any of the processes started could die ( including the process that is the script itself ) and the container may continue to live on in a busted state.
* Zombie processes,  discussed later
* We're trying to emulate (badly) what is already a solved problem, and that solved problem is init systems like runit or systemd.
* At the very least when doing this, run your final command with `exec`.  This will give the primary pid to that process.

## 43 - Init systems - Runit

* Runit is a fairly lightweight init system with very easy to write services.

* With runit installed and run as your container's entrypoint/cmd it will execute any scripts in `/etc/service/<name>/run`.  

* Usually this consists of a shell script written to run a process in the foreground and maybe set a user to run as, but it can be any executable.

* runit restarts processes on exit/crash

* runit services can be controlled by the standard service commands in linux,  and if you want to restart a process in the container you can issue a service restart command via docker exec.

* runit handles zombie processes appropriately.

## 44 - Zombies!

* unix processes are ordered in a tree with PID 1 being the top of the tree.  Each process has a parent and can have children.

* when a process dies it turns in defunct or zombie process.  Unix expects that the parent process will explicitly wait for the child process to end in order to collect its exit status.

* most applications wait for their children processes to exit correctly,  but running things from random shell scripts can play havoc with this.  If the parent process is killed, all of its children become orphans.

* the kernel deals with this problem by giving orphaned processes to PID1 / the init system and expects it to deal with them appropriately.

* Bash is actually capable of reaping child processes, but it does not handle signals properly nor does it pass those signals down to its children.   thus bash cannot be used /reliably/ as a lightweight init system.

* This /is/ a problem that can be solved with bash, and I had started working on solving it myself when I found the blog post about how phusion images handle things.

* They wrote a very lightweight init system called my_init which in turn loads runit, or a list of provided apps.   I'm not 100% sure if they solve something that needs to be solved when just running runit, or if they just use my_init first for consistency.

## 45 - Agenda slide

## 46 - Container Centric OSes

* CoreOS - the oldest most mature Container centric OS of the bunch.
* Atomic - Redhat's minimal container OS
* Snappy - Ubuntu Core - Ubuntu's
* RancherOS - recent addition to the lineup from the folks at rancherlabs.

## 47 - CoreOS

CoreOS is a free linux operating system.  It is provided under an Apache2.0 license and the source is freely available on github.   

They do offer a support model that gives access to some premium features as 'CoreOS Managed Linux' which offers some tooling around managing fleets of CoreOS servers and update management.  

CoreOS also has an enterprise registry which is priced quite reasonably and gives you an amazing behind the firewall docker registry.

At its heart CoreOS is a fork of Google's ChromeOS and is focussed at being a very lightweight server OS.

It uses FastPatch for updates, which is a active/passive root partition scheme, the OS is updated as an entire unit on the passive partition and then the partitions are simply switched and rebooted to update.

CoreOS philosphy is to patch frequently and to focus on writing applications / architectures that can handle frequent rebooting on individual nodes.  They have a series of update options that use ETCD to help enforce rules about how many systems can be patching/rebooting at a given time.

Updates are available to all users,  paying customers get access to CoreUpdate which is a hosted dashboard to help manage updates across a fleet of systems.

There is no package manager,  instead a given binary is either part of the OS itself or is completely the responsibility of the user.  It is generally expected that anything not part of the OS itself is going to run inside a container.

CoreOS comes with some tooling to help support native clustering of your hosts.  more on this in a minute.

Rocket is a new containerization technology that CoreOS has recently announced. At this time I no real experience with it beyond reading some documentation and internet opinions.

## 48 - CoreOS - Clustering

CoreOS comes with a set of tools designed to help you with clustering CoreOS hosts and applications running on them.

### etcd

etcd is a distributed, consistent key value store  written in Go for shared configuration and service discovery using the Raft consensus algorithm with a focus on being:

* Simple: curl'able user facing API (HTTP+JSON)
* Secure: optional SSL client cert authentication
* Fast: benchmarked 1000s of writes/s per instance
* Reliable: properly distributed using Raft

### fleet

fleet ties together systemd and etcd into a distributed init system. Think of it as an extension of systemd that operates at the cluster level instead of the machine level. 

A fleet object looks exactly like a systemd service definition with some additional fields to handle scheduling hints etc.

### flannel

flannel (originally rudder) is an overlay network that gives a subnet to each machine for use with Kubernetes.  It can be used to give each host's docker0 interface a set of real IPs and manages routing between the hosts.   It is not turned on by default in coreos.

## 49 - agenda

## 50 - app configuration

Unless your application is a mythical 12factor application that is configured by environment variables you'll need to figure out how to configure your application in the container.

## 51 - in dockerfile

If your application config is very static,  you can put the config file in the same directory as your Dockerfile and simply use ADD in the Dockerfile to put the config file in the correct location,  You might want to use this for say an apache container that hosts only static content that is always found in /var/www/html

## 52 - via volume mount

You can keep your config files on your hosts ( maybe placed there by configuration management ) and then bind mount them into the container.  

This is not a bad way to go if you're treating docker as a packaging format and you don't mind exposing parts of the hosts filesystem to the container.

The usual security implications of doing a bind mount apply.

## 53 - start script

If you have a config file with very few changes,  you can create a simple templated pattern of keywords in the config file like xxxPORTxxx and add it in during the docker build process.

You also have a startup script that you add in that runs an inline sed to change those keywords into the values of environment variables before running your actual command with an `exec` line at the bottom of the script.

## 54 - using confd

This is my personal favorite way of doing handling config files in containers.

Confd is is a lightweight templating engine that takes values from a key value store such as etcd or consul.  It also supports environment variables.   

## 55 - using confd

As shown here you can download a binary distribution of confd from github and mark it executable in your Dockerfile.  This shows grabbing etcd as well, this isn't essential for confd to work,  but it is cheap.

## 56 - using confd

This is an example startup script that you run in your docker container.  You can see it takes an ENV variable called `backend` and if that doesn't exist defaults to `env`.

It then attempts to write out any templates you have stored in `/etc/confd`.  I have this in a loop so that if the key-value store isn't read yet, it will just keep looping here.   Very useful if you need to wait for another service to start first and register itself in etcd, but not really necessary for environment variables.

Once confd has written the templates out it will run apache in foreground via the final `exec` command.

## 56 - using confd

Assuming you have templated the apache port and apache root you would call the apache container like this to set those values.

## 57 - using confd

We'll see more confd magic as we go through the demo shortly.

## 58 - agenda

## 59 - DEMO

### factorish/example

So we're going to do a demo and talk through some of these concepts.

Let's take a look at this example Docker repo - https://github.com/factorish/factorish/tree/master/example

* talk through dockerfile,  pointing out confd and etcd, runit.

* talk through the contents of bin/
    - boot
    - healthcheck
    - my_init

* show confd and templates directory and talk about how confd uses those dirs to write templates out.  use `confd/example.conf.toml` and `confd/example.conf` to illustrate.

### factorish

* Vagrantfile
* factorish.rb
* factorish.yml
* user-data.erb
* registry and image caching
* profiles.d scripts, how they're generated etc.
* build / fetch scripts

### factorish/elk

talk through how it works,  show fleet units, etc.
