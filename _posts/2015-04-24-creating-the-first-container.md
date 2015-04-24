---
layout: post
title:  "Creating the first Container on a Server"
date:   2015-04-24 16:00:00
categories: jenkins container dockerfile
banner_image: docker-jenkins-git.png
comments: true
author_name: Steffi, Marius, Markus
---

This post will explain how you can setup a Jenkins job that will build a Docker image which is defined by a dockerfile. Later, it will be deployed onto a remote server, so our application will be running in an own container on its own machine. 

# Our Project and the Dockerfile
In this section, we demonstrate two different approaches to build a docker container. The first one makes use of the [Docker Hub](https://hub.docker.com). Docker Hub is a platform where you can store and publish predefined docker images. Those images can downloaded by other users and executed or used as fundament for future images, respectively. The other approach is to create a Dockerfile where an image will be created based on a plain Linux image like Ubuntu. A Dockerfile is a file consisting of a sequence of various Docker command which describes the process of building an specific image.

### Using Docker Hub
As our project is written in *python* and using the Django framework, we defined a docker image containing all required dependencies like *python*, *Django*... and submitted it to Docker Hub. This image will be downloaded and processed by the following *Dockerfile*.

    FROM django:python2-onbuild 
    RUN pip install --upgrade pip
    
Other dependencies can be found in the file *requirements.txt*, so we are able to use Python dependencies such as *numpy* or *matplotlib*. 

### Creating an image based on a plain image using a Dockerfile

Now, we demonstrate how to build an image based on a plain Linux by using a Dockerfile which comprises the entire image specification. The comments provided in the Dockerfile should suffice as description for the used commands.

    # Set the base image to Ubuntu
    FROM ubuntu
    
    # File Author / Maintainer
    MAINTAINER ComputerScienceAndMedia
    
    # Update the sources list
    RUN apt-get update
    
    # Install basic applications
    RUN apt-get install -y tar git curl nano wget dialog net-tools build-essential pkg-config
    
    # Install Python3 and required dependencies
    RUN apt-get install -y python3 python3-dev python3-pip libfreetype6-dev
    
    # Adding the application and requirements to the image
    ADD /app /test_application
    ADD /requirements.txt /test_application/requirements.txt
    
    # Get pip to download and install requirements:
    RUN pip3 install -r /test_application/requirements.txt
    
    # Expose ports
    EXPOSE 8000
    
    # Set the default directory where CMD will execute
    WORKDIR /test_application
    
    # Set the default command to execute when creating a new container
    # to change the port use: python3 manage.py runserver 0.0.0.0:1234
    CMD python3 manage.py runserver

To generate an image based on this Dockerfile we execute the following command:
    
    docker build -f OurDockerfile -t my-image-name .

Please note that the command ends with a dot. This is the working directory which is used when the Dockerfile is processed. So, all paths in the Dockerfile will be based on this path. *Example*: If you set a path to `/` in the Dockerfile it will be the current working direcotry if `.` is specified in the build command.

*Note: The example in this section is adapted to our sample application. It is supposed, that the current working directory contains the entire Git repository [django_project](https://github.com/learning-continuous-deployment/django_project).*

# Setup Jenkins  
* Create a new Jenkins job as described in one of our previous posts ["How to trigger a Jenkins build process by a GitHub push"](http://learning-continuous-deployment.github.io/jenkins/github/2015/04/17/github-jenkins/).
* With installing the Docker plugin on Jenkins you will have more options for your build process. As we want to trigger a build with a new push from GitHub we tick the box "Build when a change is pushed to GitHub". But you could also predefine a time schedule. Furthermore the Docker plugin allows to build when a new image is built on DockerHub.
* If you want to test whether your container is working at all, you could try starting it through a shell first with `docker build -f "$WORKSPACE/Dockerfile" -t "csm_$BUILD_ID" $WORKSPACE`. Commands with a `$`in front are environment variables from Jenkins. `f`indicates the file name which can be found in our project workspace. `t` stands for the repository name for the image. The Docker commands can be found [here](https://docs.docker.com/reference/commandline/cli/).
* Instead using the shell we would like to use the provided Docker plugin functions such as "Execute Docker Container", which will be described here *soon*.  

##Troubleshooting
1. Avoid spaces in your Jenkins project name.
2. If you want to change the language in Jenkins simply change it in your browser. For Firefox use e.g. the add-on [Quick Locale Switcher](https://addons.mozilla.org/en-US/firefox/addon/quick-locale-switcher/). 
3. If you want to start a docker container through a shell script, you need to be an administrator, however `sudo` will not work. So it is required to create a docker group if it does not already exist.
  * `sudo groupadd docker`
  * Add the connected user `${USER}` to the docker group. `sudo gpasswd -a ${USER} docker`
  * Restart the Docker daemon with `sudo service docker restart` (for Ubuntu 14.04 use `sudo service docker.io restart`)
  * Restart your Jenkins server. 
  * Or even the whole server