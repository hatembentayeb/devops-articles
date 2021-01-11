## Hello Docker 😄 —Part I


> In this story i will share my docker experience, so, be ready ! to know the most used container technologies in the hole world ! . I will help you to get started in docker also we will deep dive into cool features … , and by the end of this story you will be able to make your own projects using docker and docker-compose 😅.
> The second part is ready : [Learn Docker From Scratch](/@hatemtayeb2/hello-docker-part-ii-998472c6e296)

# **What is Docker ?**

In a simple way … Docker is a way to bundle an app and it’s requirements into a single Image that can be runnable in any environment like _Windows_, _Linux_, _Mac_ _os_ and of course the cloud.

A docker Image is a micro os like _ubuntu_, _debian_ and _archlinux_ …, But what is a micro os ? simply it’s a very tiny os that contains only a file system and some basic command and of course a package manager like _apt_ and _pacman_ …

# What i need to get started ?

I am not going through the installation process or explaining the functional part of docker … i will go through examples from real life and some cool tips and tricks .

*   Docker installation : [Clik here](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
*   Docker vs virtual machine : [Click here](https://www.backblaze.com/blog/vm-vs-containers/)

> You must READ them 😠

Check your docker status by running this command :

> $ sudo systemctl status docker

If docker is not running make sure to run this command

> $ sudo systemctl start docker

# Running your first container

After installing docker, now you are able to follow this article, so … let’s run our first container by running this command and i will explain every step 😃

> $ docker run hello-world

This is the output :

![Image for post](https://miro.medium.com/max/60/1*nTWq4BuE0WjV2XKDpyFO2w.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2732/1*nTWq4BuE0WjV2XKDpyFO2w.png" width="1366" height="768" srcSet="https://miro.medium.com/max/552/1*nTWq4BuE0WjV2XKDpyFO2w.png 276w, https://miro.medium.com/max/1104/1*nTWq4BuE0WjV2XKDpyFO2w.png 552w, https://miro.medium.com/max/1280/1*nTWq4BuE0WjV2XKDpyFO2w.png 640w, https://miro.medium.com/max/1400/1*nTWq4BuE0WjV2XKDpyFO2w.png 700w" sizes="700px"/></noscript>

By this command we tell the docker engine to run an image called “hello-world”, which is an image that contains a simple script with the output that begins with “ Hello from docker …” so … what happens here ?

The docker engine searches for this image in the local registry but he tells us that he didn’t find it so it goes to a global registry which is a huge and public place for free images (created by other peoples ) called [_docker hub_](https://hub.docker.com/)_._

He pull or download the image then he run it, and finally we get the message or the execution result of the script inside the image. To download only the image we use _’’docker pull hello-world’’._

Let’s try another one , in this case i have an ARCHLINUX based os and i want to run an alpine system with docker and make some work on it so let’s go 😄

> docker pull alpine:latest
> 
> docker run -it alpine sh

Output :

![Image for post](https://miro.medium.com/freeze/max/60/1*l0scPgwy2PoRydrFtNgVmQ.gif?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2724/1*l0scPgwy2PoRydrFtNgVmQ.gif" width="1362" height="695" srcSet="https://miro.medium.com/max/552/1*l0scPgwy2PoRydrFtNgVmQ.gif 276w, https://miro.medium.com/max/1104/1*l0scPgwy2PoRydrFtNgVmQ.gif 552w, https://miro.medium.com/max/1280/1*l0scPgwy2PoRydrFtNgVmQ.gif 640w, https://miro.medium.com/max/1400/1*l0scPgwy2PoRydrFtNgVmQ.gif 700w" sizes="700px"/></noscript>

We have downloaded the alpine image and we run it interactively by adding “ -it” and “sh” at the end :

*   **-i** : for the interactive mode with `-t` option ( we can give input and type some commands to interact with the alpine system)
*   **-t** : Allocate a pseudo-TTY (PTY) to emulate a hardware terminal we used often with ssh and telnet protocols .
*   **sh** : is the default shell ( like bash, csh and zsh …)

The other commands ( ls , cat …) are a basic commands that are founded on any other Linux systems.

# Commit a docker container

What is the difference between a docker image and a container ? Simply the docker image is the basic resource and it can be shared by multiple containers but how ? when we make a _“docker run”_ , the docker engine make a snapshot or a copy of the base image in the memory and we use it like we did with alpine ,after some work on the snapshot or the container we usually stop and exit the container, and every container is identified with a unique hash (ID),

To add the changes that we make on the snapshot to the base image we use the “_commit_” command to persist our changes so let’s try out :

Watch this demo :

![Image for post](https://miro.medium.com/freeze/max/60/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2724/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif" width="1362" height="696" srcSet="https://miro.medium.com/max/552/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif 276w, https://miro.medium.com/max/1104/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif 552w, https://miro.medium.com/max/1280/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif 640w, https://miro.medium.com/max/1400/1*tqZ0yh-yzAjZpj2uQAxaWQ.gif 700w" sizes="700px"/></noscript>

docker commit

We have created a new image named “_alpine_modified”_ that contains a _“hello.sh”_ fileunder the “_/home”_ folder _._ You surely notice that the original modification in the alpine image is gone now.

# Create your own image

Another way to save changes or create some stuff in containers is the _“Dockerfile” ,_ which is a a set of instructions that define the final state of your image .

This file uses YAML syntax which is easy to understand and so clean 😄, so let’s go writing our first _Dockerfile_ :

dockerfile

In this file we have two keywords :

*   **FROM :** this is the base image that we use .
*   **CMD :** the Command keyword , this command will run the _“echo hello”_ when we use the _“docker run <container name> “._

let’s build the image :

> $ docker build -t hello .

*   **build** : will make the build process by executing every instruction (step)
*   **-t** : giving a name to our image
*   **.** : our build context which we have the “_Dockerfile”,_ or you can specify the hole path.

> **note**: don’t forget the dot “.” please !!

Watch the demo :

![Image for post](https://miro.medium.com/freeze/max/60/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2724/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif" width="1362" height="696" srcSet="https://miro.medium.com/max/552/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif 276w, https://miro.medium.com/max/1104/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif 552w, https://miro.medium.com/max/1280/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif 640w, https://miro.medium.com/max/1400/1*f4VZ2Mno9P0CfzsZUqdGqQ.gif 700w" sizes="700px"/></noscript>

hello_alpine

To get all images try to run : `docker images` , if you want to get more information about your image run this command : `docker inspect hello` and try to figure out what happens here , i will cover this topic in the next stories … .

Finally, this tutorial is just an introduction to docker for beginners, in the next parts i will talk about more advanced features in docker. If you have feedback feel free to share them with me [Hatem Ben Tayeb](https://www.linkedin.com/in/hatembentayeb/).

If this post was helpful click the clap button as much as possible 😃.
The second part is ready : [Learn Docker From Scratch](/@hatemtayeb2/hello-docker-part-ii-998472c6e296)

# Thank you 😃

