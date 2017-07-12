# Docker get started

These notes and related files orignate from the [Docker Get Started](https://docs.docker.com/get-started/) tutorial.

## Part 1: Orientation

An **image** is a stand-alone executable package that includes everything needed to run a piece of software, including the code, a runtime, libraries, environment variables and configuration files.

A **container** is a runtime instance of an image - what the image becomes in memory when actually executed. It runs completely isolated from the host environment by default, accessing host files and ports if configured to do so.

To check **Docker** is installed and running ok run `docker run hello-world`.
