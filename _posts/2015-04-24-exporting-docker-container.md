---
layout: post
title: "Exporting Docker images to a remote machine"
date: 2015-04-24 19:00:00
categories: docker images dockerfile
banner_image: export-docker.jpg
comments: true
author_name: Marius
---

In this post, we will explain how to export a Docker image, transfer it to a remote machine and run a Docker container based on it. This post is related to
our project and the image which is created described in a previous post.

 <!--more-->

# Exporting a Docker image

Docker provides the possibility to save images to a *tar* archive. For that purpose the command `save` is provided. The following text snippet is an excerpt of Docker's documentation.

    Usage: docker save [OPTIONS] IMAGE [IMAGE...]
    Save an image(s) to a tar archive (streamed to STDOUT by default)
        -o, --output=""    Write to a file, instead of STDOUT
         
If we assume that the image we want to export is named *myimage* the command would be one the following:

    docker save -o exportedImage.tar myimage
	
or

    docker save myimage > exportedImage.tar


By doing this, we stored the image in the `exportedImage.tar` archive. Now we can transfer this to a remote machine.

# Transferring the image to a remote server using SCP

SCP is based on the SSH protocol, so to be able to use it, we need valid credentials to log in into the remote machine. In our example, we are using an authentication by a public-private keypair. This key pair is generated by the command `ssh-keygen -t rsa -C "Jenkins"` and creates the public key file `~/.ssh/id_rsa.pub`. The content of this file (the public key) needs to be entered in the `~/.ssh/authorized_keys` file on the remote machine.

Once this is done it is possible to copy files by SCP. The command syntax is the following:
   
    scp myFile username@hostname:/target/directory

It is possible that you have to validate the fingerprint of the remote server, if you are connecting to the remote machine for the first time.

Now copy the exported Docker image in an arbitrary directory.

# Loading and starting a Docker image on a remote machine

To import or load, respectively, a Docker image into Docker use the load command. Here is another snippet from the Docker documentation:

    Usage: docker load [OPTIONS]
    Load an image from a tar archive on STDIN
        -i, --input=""     Read from a tar archive file, instead of STDIN

Load the image by executing the following command:

    ssh username@hostname "docker load < /any/directory/myimage.tar"

To check that the image was successfully loaded you can execute `docker images`. This shows a list of all available images.

Now we are ready to start our exported image!

Note: The following commands are related to the Docker image created in the [Creating the first Container on a Server](http://learning-continuous-deployment.github.io/jenkins/container/dockerfile/2015/04/24/creating-the-first-container/) post.

We can start the image by executing the `docker run` command. *Please keep in mind that we have to stop running Django server before starting a new one because we use always the same port. How you can stop a container will be explained in the appendix at the end of this post.* 

    ssh user@hostname "docker run -d -p 80:1234 myimage python3 manage.py runserver 0.0.0.0:1234"
    
This command will start our Django server on port 1234 inside the Docker container but is bound to port 80 of the host system. The `-d` flag detaches the process from the terminal, so it runs in the background and will not be terminated if we close the terminal.

And that's it. We exported a Docker image, transferred it to a remote machine and remotely started a new container based on this image.

# Appendix

If you want to stop a Docker container, because you want to stop a server provided by a container, the `docker stop` command can be used. This command requires the container's id that should be stopped. If you want to stop all running containers, simply execute the following command: `docker stop $(docker ps -a -q)`