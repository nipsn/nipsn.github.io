+++
title = "On deploying a JupyterHub instance (WIP)"
date = "2023-09-22T20:34:14+02:00"
author = "nipsn"
authorTwitter = "" #do not include @
cover = ""
tags = ["JupyterHub", "Jupyter", "docker", "AWS"]
keywords = ["JupyterHub", "Jupyter", "docker", "AWS"]
description = "A quick overview of how I built a Docker based JupyterHub instance on AWS."
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
+++

## Introduction

Feel free to skip to the TLDR at the end of the post to see how the final configuration looked like.

A few weeks ago I was tasked to build a small platform that could host some academic content for programming courses. Before that, we used to give the antendees instructions on how to run a Jupyter Notebook or Lab instance on their local machines and hand out the content to them either via email or on site, but most of the time this was to our detriment, since there was always some user who either didn't get the email with instructions or for any reason couldn't follow the installation instructions. This would usually involve us solving their issues one by one, and this could take up to an hour and in terms of money, that's quite a bit of it.

## Brainstorming

So we began thinking about our options and one crossed my mind: since Jupyter Notebook and Lab exist, surely there must be a way to manage individual instances of them. Shortly after, I began investigating and soon enough I found [JupyterHub](https://jupyterhub.readthedocs.io). It looked just what I needed: an aparently simple to use way to manage instances of Jupyter Notebook or Lab that took care of assigning resources by itself (spoiler: it kind of doesn't).

On the home page of the documentation they list two options: you either run it on "bare metal" (directly as a service on a Linux server) or you use Kubernetes to deploy it. The first one seemed kind of dodgy to me at first and the second option looked daunting since I had never used Kubernetes up to that point. In reality I was looking for a middle ground between those two: a way to deploy the service that made users unable to tamper with any other user's files (not even seeing them) that was also relatively easy for me to get up and running.

Enter Docker! I was able to find a half-baked configuration on [JupyterHub's Github page](). It basically included a Dockerfile for the Hub, since it pulled the instance image from docker.io . That wasn't going to cut it for me so I got to work on getting a custom Instance image set.

## The Instances

There were a couple of things we 100% needed on each instance: the ability to load the course's material and the Scala 3 kernel. I decided to start developing a custom Jupyter Lab image based on `jupyter/base-notebook`. I needed to first off install the Scala kernel dependencies, so on the dockerfile I used the `root` user to `apt install` everything needed:

```
USER root

RUN apt-get update && \
    apt-get install -y openjdk-8-jdk && \
    apt-get install -y ant curl git python3-pip && \
    apt-get clean;

USER $NB_UID
```

The `$NB_UID` user is the default user on the notebook/lab instance.

Then, for the kernel, I decided to go with [almond's](https://almond.sh/) Scala kernel. In the following snippet I install the Scala 2.12.17 and Scala 3 ones from all the availables:

```
RUN curl -Lo coursier https://git.io/coursier-cli && \
    chmod +x coursier && \
    ./coursier launch --fork almond:0.13.14 --scala 3.2.0 -- --install --id scala3 --display-name "Scala 3" && \
    ./coursier launch almond:0.13.14 --scala 2.12.17 -- --install --id scala212 --display-name "Scala 2.12" && \
    rm -f coursier;
```

In the end, we decided to just use the Scala 3 kernel so the final version of the dockerfile had the 4th line from the previous snippet removed.

With that out of the way, we just needed to add the course's material so every user can access it the first time they log into the platform. In the following snippet, I show how to clone a specific folder from a public repo, in this case [almond's example notebooks](https://github.com/almond-sh/examples):

```
RUN mkdir notebooks && cd notebooks && \
    git init && \
    git config core.sparseCheckout true && \
    git remote add -f origin https://github.com/almond-sh/examples && \
    echo "notebooks" >> .git/info/sparse-checkout && git pull origin master
```

For that, we are using a sparse checkout. If needed, we could just pull the whole repo. However, since the course's material was hosted on a private repo, our best option was to `ADD` the content as a `.tar.gz`. By doing something like `ADD content.tar.gz /home/jovyan/work/`, we create a folder called `content` inside the `work` directory, so a good solution overall.

There is another option available, but because of time contstrains I wasn't able to explore it: creating a Jupyter hook that run everytime the user logged in that would pull the most recent version of the content to their `work` directory. There is one single thing that bothers me about that approach: if we are dealing with a private repository, we would have to put a ssh key inside of every user's instance, and that's not very secure IMO, since they would have non-zero permissions on the repo. There must be another way of doing this, but I couldn't figure out how.

## The Hub

The Hub's required few changes. The original dockerfile pulls the latest `jupyterhub/jupyterhub` image and uses `pip` to install both `dockerspawner` and `jupyterhub-nativeauthenticator` (spoiler: we will change this last package later on), then runs the service based on a config file.

This config file is called `jupyterhub_config.py`, and it's where most of the service's configuration is done. You can check all the available options [here](https://jupyterhub.readthedocs.io/en/stable/reference/config-reference.html). We already had some basic configuration done from this example I found, so in my case it was just a matter of adding the stuff we needed. Of course, since we are using the `dockerspawner`, we will need to add some lines to our file:

```python
# Spawn single-user servers as Docker containers
c.JupyterHub.spawner_class = "dockerspawner.DockerSpawner"

# Spawn containers from this image
c.DockerSpawner.image = os.environ["DOCKER_NOTEBOOK_IMAGE"]

# JupyterHub requires a single-user instance of the Notebook server, so we
# default to using the `start-singleuser.sh` script included in the
# jupyter/docker-stacks *-notebook images as the Docker run command when
# spawning containers.  Optionally, you can override the Docker run command
# using the DOCKER_SPAWN_CMD environment variable.
spawn_cmd = os.environ.get("DOCKER_SPAWN_CMD", "start-singleuser.sh")
c.DockerSpawner.cmd = spawn_cmd

# Connect containers to this Docker network
network_name = os.environ["DOCKER_NETWORK_NAME"]
c.DockerSpawner.use_internal_ip = True
c.DockerSpawner.network_name = network_name

# Mount the real user's Docker volume on the host to the notebook user's
# notebook directory in the container
c.DockerSpawner.volumes = {"jupyterhub-user-{username}": notebook_dir}
```

Then again, this lines were already provided for me.

With that out of the way, let's see how to tie all of this together on docker-compose.

## Getting everything working together (part 1)

I started from the provided `docker-compose.yml` file, which had the configuration for the `hub` service already done, so I didn't have to put much work into it. The `instance` service however was being directly pulled from docker.io. Since I didn't really want to pull an image, but rather build one locally from a dockerfile, I decided to add the following lines:

```
services:
 instance:
    build:
      context: .
      dockerfile: Instance.dockerfile
    image: instance
    container_name: instance

 hub:
    ...
```

Here, we specify that there should be a `Instance.dockerfile` in the current context (`.`), which we will call `instance`. Simple stuff.

Now, having the following file structure:

```
├── docker-compose.yml
├── Dockerfile.jupyterhub
├── Instance.dockerfile
└── jupyterhub_config.py
```

We can do a `docker-compose build` followed by a `docker-compose up`. Aaaand it's working! Time to look for a place to host these services.

## Testing AWS

## Getting everything working together (part 2)

## Stress testing

## Getting everything working together (part 3)

## Getting HTTPS

## Deploy the monster!

## Conclusion and TLDR