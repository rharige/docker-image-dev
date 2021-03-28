## Docker images for Vespa development

This repo contains Docker images for Vespa development on CentOS 7.
[vespa-build-centos7](https://hub.docker.com/repository/docker/vespaengine/vespa-build-centos7)
is used for only building Vespa, while
[vespa-dev-centos7](https://hub.docker.com/repository/docker/vespaengine/vespa-dev-centos7)
is used for active development of Vespa with building, unit testing and running of system tests.
vespa-dev-centos7 depends on vespa-build-centos7. To pull the images:

    docker pull docker.io/vespaengine/vespa-build-centos7:latest
    docker pull docker.io/vespaengine/vespa-dev-centos7:latest

Commits to master will automatically trigger new builds and deployment on Docker Hub.

Read more at the Vespa project [homepage](http://docs.vespa.ai).

The project is covered by the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).


## Vespa development on CentOS 7

This guide describes how to build, unit test and system test Vespa on CentOS 7 using Docker.
When doing Vespa development it is important that the turnaround time between code changes
and running unit tests and system tests is short.
[vespa-dev-centos7](https://hub.docker.com/repository/docker/vespaengine/vespa-dev-centos7)
provides a complete environment for this.
The code is compiled using mvn, cmake and make and then installed into your personal install directory.
Vespa can be executed directly from this directory when for instance running system tests.


### Docker configuration

#### Docker on macOS

Make sure Docker has sufficient resources:

Open Docker - Preferences - Resources and set:
* CPUs to minimum 2. Use 8 or more for faster build times.
* Memory size to minimum 8 GB. 16 GB is preferred.
* Disk image size to 128 GB.

#### Docker on Linux

Make sure Docker can be executed without sudo for the scripts in this guide to work:

    sudo groupadd docker
    sudo usermod -aG docker $(id -un)
    sudo systemctl restart docker
    newgrp docker

Perform a new ssh login to the host.


### Setup the Docker container

#### Download the latest vespa-dev-centos7 Docker image

    docker pull docker.io/vespaengine/vespa-dev-centos7:latest

#### Create the Docker container

##### With explicit Docker volume (recommended for macOS)

First, create a long lived Docker volume.
This lets us persist data generated by and used by the Docker container.
Skip this step if the volume already exists.

    docker volume create volume-vespa-dev-centos7

Second, create the container by mounting the volume as the home directory inside the container:

    docker create \
        -p 127.0.0.1:3334:22 \
        -v volume-vespa-dev-centos7:/home/$(id -un) \
        --privileged \
        --name vespa-dev-centos7 \
        docker.io/vespaengine/vespa-dev-centos7:latest

##### With directory volume mount (recommended for Linux)

A directory on the host machine can be mounted into the container using the -v option.
This lets us persist data generated by and used by the Docker container.
When running Docker on a Linux host there is basically no overhead doing so.
First, create a volume directory on the host:

    mkdir -p $HOME/volumes/vespa-dev-centos7

Second, run docker create with the -v option to mount the volume directory as the home directory in the container:

    docker create \
        -p 127.0.0.1:3334:22 \
        -v $HOME/volumes/vespa-dev-centos7:/home/$(id -un) \
        --privileged \
        --name vespa-dev-centos7 \
        docker.io/vespaengine/vespa-dev-centos7:latest

#### Start the Docker container

    docker start vespa-dev-centos7

#### Configure the Docker container

Ensure you have an SSH key before running the configure-container.sh script.
If not, use the following guide
[to generate a new SSH key](https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).

    mkdir -p $HOME/git
    cd $HOME/git
    git clone git@github.com:vespa-engine/docker-image-dev.git
    cd $HOME/git/docker-image-dev/dev/centos7
    ./configure-container.sh vespa-dev-centos7

This adds yourself as user in the container, copies authorized keys to ensure ssh can be used,
and sets environment variables needed for building Vespa.

#### Build the vespa-dev-centos7 Docker image (optional)

    cd $HOME/git/docker-image-dev/dev/centos7
    docker build -t vespaengine/vespa-dev-centos7:latest .

Use this for testing if doing changes to the Docker image.


### Build Vespa

#### SSH into the container

    ssh -A 127.0.0.1 -p 3334

#### Checkout Vespa repo

    mkdir -p $HOME/git
    cd $HOME/git
    git clone git@github.com:vespa-engine/vespa.git

#### Build Java modules

    cd $HOME/git/vespa
    ./bootstrap.sh java
    mvn clean install --threads 1C -DskipJavadoc -DskipTests

#### Build C++ modules

    cd $HOME/git/vespa
    cmake3 .
    make -j 9

Set the number of compilation threads (-j argument) to the number of CPU cores + 1.

#### Install modules

    make install/fast

Default install directory is $HOME/vespa ($VESPA_HOME).


### Run unit tests

#### Test all Java modules

    mvn test --threads 1C

#### Test specific Java module (e.g. container-search)

    mvn test --threads 1C -pl container-search

#### Test all C++ modules

    ctest -j 9

#### Test specific C++ module (e.g. searchlib)

    ctest -j 9 -R "^searchlib_"


### Run system tests

#### Checkout system-test repo

    cd $HOME/git
    git clone git@github.com:vespa-engine/system-test.git

Note that the system test scrips are already in your PATH inside the Docker container.

#### Start nodeserver in one terminal

    nodeserver.sh

#### Run system test in another terminal

    cd $HOME/git/system-test/tests/search/basicsearch
    runtest.sh basic_search.rb


### Use CLion or IntelliJ via X11 forwarding from macOS

#### Install XQuartz for macOS
XQuartz is a version of the X.Org X Window System for macOS. Download
[here](https://www.xquartz.org/).

#### Configure sshd inside container to use ipv4
Set ```AddressFamily inet``` inside ```/etc/ssh/sshd_config``` and restart sshd:

    sudo kill -HUP <sshd-pid>

#### ssh into container with X11 forwarding
Open a XQuartz terminal and run:

    ssh -Y -A 127.0.0.1 -p 3334

Then start CLion or IntelliJ from this terminal.
