# Docker get started

These notes and related files orignate from the [Docker Get Started](https://docs.docker.com/get-started/) tutorial.

## Part 1: Orientation

An **image** is a stand-alone executable package that includes everything needed to run a piece of software, including the code, a runtime, libraries, environment variables and configuration files.

A **container** is a runtime instance of an image - what the image becomes in memory when actually executed. It runs completely isolated from the host environment by default, accessing host files and ports if configured to do so.

To check **Docker** is installed and running ok run `docker run hello-world`.

## Part 2: Containers

An app built "the docker way" has the following hieracrchy

- Stack - defines the interactions of all services
- Services - defines how container behave in production
- Container

It changes the dev environment. Before **Docker** as a dev my first task would be to install the necessary dependencies. In the tutorial this would be installing a Python runtime.

With *Docker** you can just grab the Python runtime as an image, which you can then build your own on top of.

### Dockerfile

It defines what goes on in the environment inside your container. Here is the initial example from the tutorial

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

### Build & run

Create the **Docker** image with `docker build -t friendlyhello .` (the `-t` arg allows us to tag the image with a friendly name. List your images with `docker images`

This also adds it to my 'local' registry.

Run the app with `docker run -p 4000:80 friendlyhello`. The `-p` arg maps the host's port 4000 to the container's `EXPOSE`d port 80.

To run in the background in detached mode use the `-d` argument.

`docker ps` will list all running containers, and you can stop a detached container using `docker stop 1fa4ab2cf395` (get the ID using `docker ps`).

### Share the image

A **registry** is a collection of respositories, and a **repository** is a collection of images. An *account** on a registry is used to create and manage respositories. The **Docker** CLI uses **Docker's** public registry by default, but you can ue others, and create your own private ones.

You'll need a [Docker account](https://cloud.docker.com/) before you can run `docker login`.

To publish your image to a registry you first need to associate it using `docker tag image username/repository:tag`.

```bash
docker tag friendlyhello cruikshanks/get-started:part1
```

The `:tag` argument is optional but reccommended, as its the mechanism that registries use to give Docker images a version.

You then publish it using `docker push cruikshanks/get-started:part1`.

### Use the image

From any machine that has **Docker** installed run `docker run -p 4000:80 cruikshanks/get-started:part1`. The first time this will pull down the image, and then run it.

That user has not had to install Python, or the dependencies but is running the exactly the same version of the app as you.

### Commands used and related to this section

```bash
docker build -t friendlyname .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyname  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyname         # Same thing, but in detached mode
docker ps                                 # See a list of all running containers
docker stop <hash>                     # Gracefully stop the specified container
docker ps -a           # See a list of all containers, even the ones not running
docker kill <hash>                   # Force shutdown of the specified container
docker rm <hash>              # Remove the specified container from this machine
docker rm $(docker ps -a -q)           # Remove all containers from this machine
docker images -a                               # Show all images on this machine
docker rmi <imagename>            # Remove the specified image from this machine
docker rmi $(docker images -q)             # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```

## Part 3: Services

This is all about scaling and load balancing our application.

In a distributed application different pieces of the app are called **services**.

A **service** only runs one image, but it codifies the way that image runs - what ports it should use, how many replicas of the container should run etc.

A `docker-compose.yml` file is a YAML file that defines how **Docker** containers should behave in production.

The actual name is optional, so you could have a `docker-compose-dev.yml` for example.

### Run as a load-balanced app

The first step was to run `docker swarm init` however the reasons why will coming in a later part of the guide.

To run as a load balanced app call `docker stack deploy -c docker-compose.yml getstartedlab`. You have to give your app a name.

We can then see the 5 containers we just started with `docker stack ps getstartedlab`.

When I call `curl -XGET http://localhost/` I'll see different ID's returned as it load balancer sends the request to one of the 5 containers.

To take it down `docker stack rm getstartedlab`.

### Scaling the app

To scale, change the value of `replicas` in `docker-compose.yml` and simply re-run `docker stack deploy -c docker-compose.yml getstartedlab`. Docker does an in-place update, so the stack is left as is and no containers are killed`.

### Take down

To take down the app use `docker stack rm getstartedlab`. The 'app' will be removed but the `swarm` will still exist.

To take down the swarm call `docker swarm leave --force`.

### Recasp

Calling `docker run` will get you a basic container but in production you should be thinking in terms of a **service**, i.e. the codification of a container's behaviour in a Compose file.

### Commands used and related to this section

```bash
docker stack ls              # List all running applications on this Docker host
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker stack services <appname>       # List the services associated with an app
docker stack ps <appname>   # List the running containers associated with an app
docker stack rm <appname>                             # Tear down an application
```

## Part 4: Swarms

Deploying into a cluster, running on mulitple machines is a **swarm**. A **swarm** is a group of machines running **Docker** that are joined in a cluster.

Once joined and the swarm setup, you call `docker` commands as normal but now they are executed across the cluster via a **swarm manager**. Machines in a swarm are referred to as **nodes**.

Swarm managers use several strategies to run containers for example

- **emptiest node** - least utilized machines get filled with containers
- **global** - ensures each machine gets exactly one instance of the specified container.


Strategies are set in the `compose.yml` file.

Swarm managers are machine in a swarm with the authority to execute commands, and add machines. Everything else in the swarm is a **worker**.

To become a swarm manager you tell **Docker** to enable swarm mode which makes the current machine a swarm manager.

### Setup swarm

Run `docker swarm init` to enable swarm mode and make current machine a swarm manager. Then on the machines you want to join the swarm run `docker swarm join`.

The commands specifically in the example were

```bash
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
docker-machine ssh myvm1 "docker swarm init --advertise-addr "$(docker-machine ip myvm1)
docker swarm join --token SWMTKN-1-12wvuuoc069xcqqorymn1c4jng5l0hj12eu5wvmwfjbku9d1jd-d7y56d6cguc9djtqyfc2oz6f7 192.168.99.100:2377
```

So I create two VM's with **Docker** running on them. I then make the first the swarm manager (because the VM reported multiple interfaces/IP addresses I needed to add the `--advertise-addr` arg). Then I add the second to the swarm as a worker.

You can also call `docker-machine ssh myvm2` to access a terminal session for the machine. Type `exit` when you are done.

The way of doing it shown heres saves having to log in and out of each machine.

### Deploy to the cluster

First we need to copy our `docker-compose.yml` to the swarm manager's home directory. Then we run the same command to deploy as we used in part 3.

```bash
docker-machine scp docker-compose.yml myvm1:~
docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
```

You can check the distribution of the app using `docker-machine ssh myvm1 "docker stack ps getstartedlab"`.

### Accessing the cluster

You can access the app on either `myvm1` or `myvm2`. A call to either is still load balanced across the cluster.

```bash
docker-machine ls # will also list the IP's
curl -XGET http://192.168.99.100 # port 80 has been mapped automatically
curl -XGET http://192.168.99.101
```

Both IP addresses work because nodes in a swarm participate in an ingress **routing mesh**. This ensures that a service deployed at a certain port within the swarm always has that port reserved to itself, no matter what node is actually running the container.

So if we specify our **service** runs on port 80, no matter which node it runs on, it will get port 80.

### Iterate and scale

Scaling works in the same way as before; update `docker-compose.yml` and rerun 

```bash
docker-machine scp docker-compose.yml myvm1:~
docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
```

You should also be able to change the app behaviour by editing the code and also running `docker stack deploy` again.

However it does involve going through a full cycle. So having changed `appy.py` to say *Hola* instead of *Hello* here are the commands I ran to get the change applied to my swarm.

```bash
docker build -t friendlyhello .
docker tag friendlyhello cruikshanks/get-started:part4
docker push cruikshanks/get-started:part4
# Amend docker-compose.yml to pull part4 instead of part1
docker-machine scp docker-compose.yml myvm1:~
docker-machine ssh myvm1 "docker stack deploy -c docker-compose.yml getstartedlab"
curl -XGET http://192.168.99.100
```

Possible **gotcha**! The nodes and containers are updated on a rolling basis, not in one big bang. So if you immediately start calling `CURL` you might still see *hello* being returned.

You'll also need to run `docker stack deploy` after adding new nodes to ensure the swarm takes advantage of the new resources.

### Cleanup

You can tear down the stack with `docker-machine ssh myvm1 "docker stack rm getstartedlab"`

To clear up the swarm its

```bash
docker-machine ssh myvm2 "docker swarm leave"
docker-machine ssh myvm1 "docker swarm leave --force"
```

### Commands used and related to this section

```bash
docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
docker-machine env myvm1                # View basic information about your node
docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
docker-machine start myvm1            # Start a VM that is currently not running
docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
docker-machine scp docker-compose.yml myvm1:~     # Copy file to node's home dir
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app
```

