Outline
Here’s an outline of what’s needed to run this experiment:

download a Linux ISO image
extract the kernel and initrd files from it
build the command line client vftool to launch the VM
launch the VM and connect to it
install Docker
fix an issue with aufs, which (I believe) is due to running from a live CD
None of these steps are overly complex. In fact many of them are simple copy-and-paste commands for the command line. Let’s go through them one by one.

Download the Linux ISO image
To get this to work we need an ARM Linux ISO image. I didn’t do a whole lot of research and ended up downloading an image from this link: 64-bit ARM (ARMv8/AArch64) desktop image.

Then we follow the instructions as laid out in Jacopo’s README:
```bash
sudo mkdir /Volumes/Ubuntu
sudo hdiutil attach -nomount ~/Downloads/focal-desktop-arm64.iso
sudo mount -t cd9660 /dev/diskN /Volumes/Ubuntu # see below for how to determine N
open /Volumes/Ubuntu/casper
```

In order to determine which partition number N to use, check the output of the hdiutil attach command:
```bash
❯ sudo hdiutil attach -nomount focal-desktop-arm64.iso
/dev/disk4              FDisk_partition_scheme
/dev/disk4s1            0xCD
/dev/disk4s2            0xEF
```
We’re looking for the FDisk_partition_scheme partition.

Extract vmlinuz and initrd
We have now mounted the ISO image and opened the folder with vmlinuz and initrd, which we’re looking to extract. Copy them to the folder alongside your ISO image.

In order to be able to launch the kernel image, you need to unzip it as follows:

```bash
cp vmlinuz vmlinuz.gz
gunzip vmlinuz.gz
```
### Build vftool
Next we’ll compile the command line tool to boot the vm. Clone the repository and build it:

```bash
git clone https://github.com/evansm7/vftool
cd vftool && xcodebuild
```
The binary is in build/Release and you can verify it’s running by launching it without arguments:

```bash
❯ ./build/Release/vftool
Syntax:
./build/Release/vftool 
Options are:
-k  [REQUIRED]
-a 
-i 
-d 
-c 
-b  [otherwise NAT]
-p 
-m 
```
### Launch the virtual machine
Now we’re ready to launch the Linux VM. I’ve copied vftool to the folder containing the ISO image and files vmlinuz and initrd and am running it as follows:

```bash
./vftool -k vmlinuz -i initrd -d focal-desktop-arm64.iso -m 4096 -a "console=hvc0"
```
Doing so should give you the following output:
```bash
❯ ./vftool -k vmlinuz -i initrd -d focal-desktop-arm64.iso -m 4096 -a "console=hvc0"
2020-11-26 20:18:25.753 vftool[5068:113057] vftool (v0.1 25/11/2020) starting
2020-11-26 20:18:25.753 vftool[5068:113057] +++ kernel at vmlinuz -- file:///Users/sas/Downloads/linux-vm/, initrd at initrd -- file:///Users/sas/Downloads/linux-vm/, cmdline 'console=hvc0', 1 cpus, 4096MB memory
2020-11-26 20:18:25.759 vftool[5068:113057] +++ fd 3 connected to /dev/ttys004
2020-11-26 20:18:25.759 vftool[5068:113057] +++ Waiting for connection to:  /dev/ttys004
2020-11-26 20:18:56.444 vftool[5068:113057] +++ Attaching disc focal-desktop-arm64.iso -- file:///Users/sas/Downloads/linux-vm/
2020-11-26 20:18:56.444 vftool[5068:113057] +++ Configuration validated.
2020-11-26 20:18:56.445 vftool[5068:113057] +++ canStart = 1, vm state 0
2020-11-26 20:18:56.846 vftool[5068:113446] +++ VM started
```

The important bit here is the terminal (tty) the VM is listening on: /dev/ttys004. This is how we are going to connect via screen:
```bash
screen /dev/ttys004
```
Connecting to the tty will start the VM’s boot process and you should see the familiar Linux boot sequence before arriving at the login screen:

The default user for the image I’ve linked to above is ubuntu, with no password.

## Set Datetime
```bash
sudo date --set "15 DECEMBER 2020 11:40 AM"
```

## Installing Docker
Now that we’ve logged in, we can install Docker following the usual instructions. In a nutshell:
```bash
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Note that we’re specifying [arch=arm64] here, so be careful when copy-pasting instructions, because the Docker instructions will be showing amd64 instead, for the x86 version.

For easier launching of Docker without sudo, let’s add ubuntu to the docker group and re-login:

```bash
sudo usermod -aG docker ${USER}
su - ${USER}
```
### Run Docker
We’re almost there now, just a small issue to fix with an error which you’ll see if you run a docker container now:
```bash
ubuntu@ubuntu:~$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
256ab8fe8778: Pull complete
Digest: sha256:e7c70bb24b462baa86c102610182e3efcb12a04854e8c582838d92970a09f323
Status: Downloaded newer image for hello-world:latest
[  629.331388] aufs test_add:264:dockerd[4370]: already stacked, /var/lib/docker/aufs/diff/65cc8680a79dddd4ac07ddc823f93c8de7301f58d43288fd4029041cecbd3269-init (overlay)
docker: Error response from daemon: error creating aufs mount to /var/lib/docker/aufs/mnt/65cc8680a79dddd4ac07ddc823f93c8de7301f58d43288fd4029041cecbd3269-init: mount target=/var/lib/docker/aufs/mnt/65cc8680a79dddd4ac07ddc823f93c8de7301f58d43288fd4029041cecbd3269-init data=br:/var/lib/docker/aufs/diff/65cc8680a79dddd4ac07ddc823f93c8de7301f58d43288fd4029041cecbd3269-init=rw:/var/lib/docker/aufs/diff/788ff1c3c3fac57d15c60b22ea668afe9453fd553ae23d940f7c6b9f6ad100ae=ro+wh,dio,xino=/dev/shm/aufs.xino: invalid argument.
See 'docker run --help'.
```
This error probably relates to us running off a live CD image and Stack Overflow has a work-around:
```bash
sudo sh -c "cat < /etc/docker/daemon.json
{
    "storage-driver": "vfs"
}
EOF"
sudo service docker restart
```

With this we can successfully launch a container 🎉
```bash
docker run hello-world
```
One thing to bear in mind is the following the Docker docs have to say about the vfs driver:

The vfs storage driver is intended for testing purposes, and for situations where no copy-on-write filesystem can be used. Performance of this storage driver is poor, and is not generally recommended for production use.

It shouldn’t be hard to fix this issue properly (and please get in touch if you know how!) but in the meantime, this is already helping me run tests against a Postgres database running in Docker entirely on my M1 machine – and I hope it will help you get started with Docker on Apple Silicon as well.# macbook-ubuntu
Run Ubuntu on Macbook M1

We left off at booting an ARM Linux virtual machine, and installing & running Docker inside it. This works fine and allows you run containers like for instance a Postgres database. They can expose their ports and instead of connecting to localhost:5432 on the host, you connect to your VM’s IP address, for instance 192.168.64.8:5432.

Now in case you’re not aware, Docker is actually a service that exposes an API over HTTP and when you run docker commands, they control the service via these API requests on a local socket. You’ve probably seen the reference to unix:///var/run/docker.sock at times.

What that means is that it’s possible to direct your docker client to talk to a Docker service over the network, and that’s what we can do as well to enable “regular” docker commands to work with a locally running “Linux Docker VM”.

Incidentally, that’s exactly what “Docker for Mac” is doing under the hood: it is actually a VM based on Apple’s Hypervisor framework which hosts Docker, not your Mac itself.

Installing “Docker for Mac”
In order to get started, we need to obtain the Docker client app. I migrated my existing setup from a Time Machine backup and therefore had Docker installed from the start. If you need to install from scratch, installing the client binaries is probably the easiest way to get started.

Assuming you have a docker binary to hand, you’re good to go. The way we’ll set up the connection is via a “docker context” and all this is thanks to a tip by Johannes Weiss.

Setting up SSH in virtual machine
First though, we need to configure to connect to our VM via SSH. Install the SSH server:

sudo apt-get install openssh-server
and copy your public key to the ubuntu user’s authorized_keys for password-less access.

In order to connect to the VM, you’ll need to find its IP address (which is on a 192.168.64.0 network if you’ve followed along from the previous blog post).

Our image comes without net-tools, so we’ll install these first:
```bash
sudo apt install net-tools
```
and then run
```bash
ifconfig | grep 192.168
```
which should yield the following
```bash
ubuntu@ubuntu:~$ ifconfig | grep 192.168
        inet 192.168.64.8  netmask 255.255.255.0  broadcast 192.168.64.255
```
With all this out of the way, we can test the connection from the host:
```bash
ssh ubuntu@192.168.64.8
```
Setting up a Docker context
The remaining steps are simple. We create a Docker context:
```bash
docker context create myvm --docker "host=ssh://ubuntu@192.168.64.8"
```
and tell Docker to use it
```bash
docker context use myvm
```
Now all docker commands are targeting the virtual machine, just like they do in Docker for Mac.
```bash
❯ docker context create myvm --docker "host=ssh://ubuntu@192.168.64.8"
myvm
```bash
Successfully created context "myvm"
~
❯ docker context use myvm
myvm
~
❯ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
~
❯ docker run hello-world
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
You can find out more about docker context in the online documentation.

Bear in mind that this setup does not come with the convenience of Docker for Mac’s bridge networking or support for volumes – but surely these aren’t far off.
