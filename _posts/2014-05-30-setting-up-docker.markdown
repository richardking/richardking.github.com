### What is Docker

Docker[^docker] is an open-source project that automates the deployment of applications inside software containers. You can use Chef or Puppet to try and configure a local environment a certain, very specific way, but even a small difference in the original environments (OS update, environment variables, etc) could cause issues down the road. I'm sure everyone has run into the problem of an application working in someone else's local environment but not yours.

By packaging an application up with all of its dependencies (essentially snapshotting the OS in a quick, easy and repeatable way), Docker tries to solve that problem.

### Getting started with Docker

Docker is made up of a daemon and a binary which acts as a client. It has excellent [documentation](http://docs.docker.io/), which walks you through all the steps of setting up Docker.  On all native Linux machines, the installation of the daemon is relatively straightforward. I am using a Mac, so I needed to install a workaround VM (which contains the Docker daemon) called ```boot2docker``` to get it working on my machine. I'll walk through the steps and the gotchas I ran into here.

Mac installation instructions are [here](http://docs.docker.io/installation/mac/). I ran through the **Installing VirtualBox** steps and then the **Installing boot2docker with Homebrew** steps. At the time, I ran into one small issue while installing ```boot2docker```, which may or may not exist now. It is outlined [here](https://github.com/boot2docker/boot2docker/issues/149), and I resolved it by moving ```boot2docker.iso``` into the ```bin``` dir.

The rest of the instructions to get ```boot2docker``` up are straightforward. Just three gotchas that I ran into:

1. During the initial setup, completing the **Using Docker port forwarding with boot2docker** section is optional. However, if you ever start a container as a web server, you'll need these port forward configurations to access it. This is because the container will configure a forward port (looks something like ```0.0.0.0:49158->80/tcp```), but that port number is the port on the VM, not your local machine. So you won't be able to actually access it unless you run those Docker port forwarding steps.
2. The instructions state you need to run ```export DOCKER_HOST=tcp://127.0.0.1:4243```, which lets the Docker client know where the Docker daemon is. You may want to include ```DOCKER_HOST=tcp://127.0.0.1:4243``` into your ```.bashrc``` (or equivalent) so that you don't have to run the export command every time you open a new shell.
3. If you ever have trouble connecting to the Docker daemon (if any of the ```docker``` commands gives an error), restart boot2docker (```boot2docker restart```). That resolves most of the problems for me.

### Testing your Docker installation

The next step is to run a simple command within a Docker container. This page walks you through the steps of printing out "Hello World" from within a Docker container: http://docs.docker.io/examples/hello_world/. Pay particular attention to the syntax of running a command in a Docker container:

> ```$ sudo docker run busybox /bin/echo hello world```

> This command will run a simple ```echo``` command, that will echo ```hello world``` back to the console over standard out.

> **Explanation:**

> *  "sudo" execute the following commands as user root
> * "docker run" run a command in a new container
> * "busybox" is the image we are running the command in.
> * "/bin/echo" is the command we want to run in the container
> * "hello world" is the input for the echo command

### Docker images and containers

#### Listing Docker images
So now that we have an image and container, we can take a look at some commands to view and run them commands within them. First is the command to view your local Docker images:
```
docker images
```

A Docker image is a read-only layer[^layer] that never changes. These are self-contained environments that can be passed around and will run exactly the same from machine to machine.

Similar to github, Docker has its own online public (and private) repository at http://index.docker.io. In the **Testing your Docker installation** section, when you ran ```sudo docker pull busybox```, it looks for and then downloads the ```busybox``` repository from index.docker.io. You can also push your own images onto the Docker index with ```docker push [name of image]```.

#### Listing Docker containers
Once you have an image, you can run a command on it. Any images that have had a command run within it, or any image that have a command currently running on it, can be displayed using:
```
docker ps -a
```
To view only active running Docker containers, just leave off the ```-a``` (```docker ps```).

#### Running a command in a Docker container
To access the bash shell of an image, run the image using:
```
docker run -t -i [image name/id] /bin/bash
```

The ```-t``` tells it to allocate a pseudo-tty, and ```-i``` tells it to run in interactive mode. The last part of the ```run``` command is the command you want to be run in the container, so in this case we are running ```/bin/bash``` to open up a shell. After you run that docker command, you should see a new command prompt, something that looks like:
```
root@f14a378e2186:/#
```
This ```docker run``` command will spawn a new container from the specified image. So if you open up another local terminal window and run ```docker ps```, you should see a new container listed.

#### Stopping an active Docker container
Docker containers automatically stop running if the command you pass to it completes. So running ```docker run busybox /bin/echo "hello world"``` would result in "hello world" getting printed out and then immediately afterward the container would stop running. On the other hand, a command like ```/bin/bash``` would have the container running continuously until it is manually stopped.

To manually stop a container, you run ```docker stop [container id]```.

#### Removing a Docker container
All stopped containers will continue to exist, so you can restart them at any point using ```docker start [container id]```. To completely remove a container, you can run ```docker rm [container id]```.
### Dockerfile


### Linking containers

[^docker]: Full background at https://www.docker.io/the_whole_story/

[^layer]: http://docs.docker.io/terms/layer/#layer-def