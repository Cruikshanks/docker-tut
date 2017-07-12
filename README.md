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

Create the **Docker** image with `docker build -t friendlyhello` (the `-t` arg allows us to tag the image with a friendly name. List your images with `docker images`

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

