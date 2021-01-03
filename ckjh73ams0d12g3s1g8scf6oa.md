## Optimizing CI/CD Pipeline for Rust Projects (Gitlab & Docker)

<span class="s"></span>![Image for post](https://miro.medium.com/max/60/1*OEM_jqCzqcbfSNXyzRqtAA.png?q=20)



<noscript><img alt="Image for post" class="t u v du aj" src="https://miro.medium.com/max/5464/1*OEM_jqCzqcbfSNXyzRqtAA.png" width="2732" height="1536" srcSet="https://miro.medium.com/max/552/1*OEM_jqCzqcbfSNXyzRqtAA.png 276w, https://miro.medium.com/max/1104/1*OEM_jqCzqcbfSNXyzRqtAA.png 552w, https://miro.medium.com/max/1280/1*OEM_jqCzqcbfSNXyzRqtAA.png 640w, https://miro.medium.com/max/1456/1*OEM_jqCzqcbfSNXyzRqtAA.png 728w, https://miro.medium.com/max/1632/1*OEM_jqCzqcbfSNXyzRqtAA.png 816w, https://miro.medium.com/max/1808/1*OEM_jqCzqcbfSNXyzRqtAA.png 904w, https://miro.medium.com/max/1984/1*OEM_jqCzqcbfSNXyzRqtAA.png 992w, https://miro.medium.com/max/2160/1*OEM_jqCzqcbfSNXyzRqtAA.png 1080w, https://miro.medium.com/max/2700/1*OEM_jqCzqcbfSNXyzRqtAA.png 1350w, https://miro.medium.com/max/3240/1*OEM_jqCzqcbfSNXyzRqtAA.png 1620w, https://miro.medium.com/max/3780/1*OEM_jqCzqcbfSNXyzRqtAA.png 1890w, https://miro.medium.com/max/4320/1*OEM_jqCzqcbfSNXyzRqtAA.png 2160w, https://miro.medium.com/max/4800/1*OEM_jqCzqcbfSNXyzRqtAA.png 2400w" sizes="100vw"/></noscript>

Rust language

# Optimizing CI/CD Pipeline for Rust Projects (Gitlab & Docker)


> Before we begin, i would like to thank [Astrolab-Agency](https://astrolab-agency.com/) for the Internship opportunity and for their trust on me to make this project, i would like to thank Mr [Mahdi Ben Chikh](https://www.linkedin.com/in/mehdi-ben-cheikh-b8b3855a/) for his precious support along the intern period.

# **What is Rust ?**

Rust is a programming language ( general purpose) C-like, which mean it is a compiled language and it comes with new strong features in managing memory and more. The cool thing ! rust does not have a garbage collector and that is _awesome_ 😅 .

# **What is DevOps ?**

In short, Devops is the key feature that helps the **dev** team and the **ops** team to be friends 😃 without a work conflicts , It is the ART of automation. It increase the velocity of delivering a better softwares !

# Identifying the problem

we can make a lot of things with rust like web apps , system drivers ans much more but there is one problem which is the time that rust takes to make a binary by downloading dependencies and compile them.

The **cargo** command helps us to download packages ( crates in the rust world) , The **Rustc** is our compiler. Now we need to make a pipeline using the Gitlab CI/CD and docker to make the **_deployment faster_**.

> This is our challenge and the Goal of this article ! 👊

# **Static linking Vs Dynamic linking**

Rust by default uses a Dynamic linking method to build the binary, so what is _dynamic linking_ ?.

The Dynamic linking uses shared libraries , so the lib is loaded into the memory and only the address is integrated into the binary. In this case the `libc` is used.

The Static linking uses static libraries which is integrated physically into the binary, no addresses are used and the binary size will be more bigger. In this case the `musl libc` is used.

You want to know more ? Then check this : [click here](/@dkwok94/the-linking-process-exposed-static-vs-dynamic-libraries-977e92139b5f).

# Optimizing the CI/CD pipeline

The CI/CD pipeline is a set a steps that allow us to make :

> build → test → deploy

In this article i will focus on the build stage because in my opinion it is very sensitive phase and it will affect the “Time to market” approach !

So the first thing is to optimize the size of our docker images to make the deployment faster. Before we begin, i will use a simple rust project for the demo.

![project structure](https://miro.medium.com/max/60/1*z2-pMILSKWMFE9oGAV3mPA.png?q=20)

<noscript><img alt="project structure" class="t u v du aj" src="https://miro.medium.com/max/2732/1*z2-pMILSKWMFE9oGAV3mPA.png" width="1366" height="768" srcSet="https://miro.medium.com/max/552/1*z2-pMILSKWMFE9oGAV3mPA.png 276w, https://miro.medium.com/max/1104/1*z2-pMILSKWMFE9oGAV3mPA.png 552w, https://miro.medium.com/max/1280/1*z2-pMILSKWMFE9oGAV3mPA.png 640w, https://miro.medium.com/max/1400/1*z2-pMILSKWMFE9oGAV3mPA.png 700w" sizes="700px"/></noscript>

The project structure

let’s understand the project structure :

*   **src** : This dir contains all source code of the app (***.rs** files).
*   **Cargo.toml** : This file contain the package meta-data and the dependencies required by the app and some other features … .
*   **Cargo.lock** : Ct contains the exact information about your dependencies.
*   **Rocket.toml** : With this file we specify the app status ( development , staging or production) and the required configuration for each mode, for example the port configuration for each environment.
*   **Dockerfile** : This is the docker file configuration to build the image with the specific environment that is configured already in _Rocket.toml._

Are you prepared 👊 😈 !!! , let’s begin the show !! 🎉 🎉 🎉

We will begin by building the app image locally , so let’s see how the docker file look like :

This Dockerfile is splitted into two sections :

*   The builder section ( a temporary container)
*   The final image (Reduced in size)

**The builder section:**

In order to use rust we have to get a pre-configured images that contains the Rustc compiler and the Cargo tool. the image have the rust nightly build version and this is a real challenge because it’s not stable 😠.

> We will use the static linking to get fully functional binary that doesn’t need any shared libraries from the host image !!

let’s breakdown the code :

*   First we import the base image.
*   We need the _MUSL_ support : `musl-tool` after updating the **source.list** of your packages `apt-get update` , _MUSL_ is an easy-to-deploy static and minimal dynamically linked programs.
*   Now we have to specify the target , if you don’t know ! no problem ! you can use `x86_64-unknown-linux-musl` , run with Rustup (_the rust toolchain installer_)
*   To define the project structure on the container we use `cargo new --bin material` (material is the project name), it’s much like the structure that we see earlier.
*   Making the `material` directory as a default we use the `WORKDIR` Dockerfile command.
*   The `Cargo.toml` and `Cargo.lock` are required for deps. installation
*   Setting up the `RUST_FLAGS` with `-Clinker=musl-gcc` : this flag tell cargo to use the musl gcc to compile the source code , the `--release` argument is used to prepare the code for a release ( _final binary optimization_).
*   `--target` specify the target compilation 64 or 32 bit
*   `--feature vendored` thsi command is an angle 😄 ! it helps to solve any ssl problem by finding the SSL resources automatically without specifying the SSL lib directory and the SSL include directory. It saves me a lot of time, this command is associated with some configurations in the Cargo.toml file under the `feature` section.

> Until now we only build the dependencies in **_Cargo.toml_** and we make **the clean** ( removing unnecessary files)

*   After downloading and compiling required packages, it’s the time to get the source code into the container and make the final build to produce the final binary ( **standalone**).

The builder stage has complete ! congrats 😙 🎉 yeah !!. Now let’s use alpine as a base image to get the binary from the build stage , but ! wait a second ! what is alpine ???

> Alpine is a Linux distribution, it’s characterized in the docker world by his size ! it is a very small image (4MB) and it contains only the base commands (busybox)

*   `--from=cargo-build ..../material` now we will copy the final binary to the alpine and the intermediate container (cargo-build) will be destroyed and we get as a result a very tiny image (12–20MB) ready to use 😃 😃 😃

> You know how to build a docker image right 😲 ? okay 😃

# The CI/CD pipeline

After testing the image locally, it seems good 😃, we resolve the docker image size, but in CI system the velocity is very important than size !! so let’s take this challenge and reduce the compilation time of this rust project !!

let’s look at the **.gitlab-ci.yml** file ( _our CI confi_guration):

There is a tip in this file , i just splitted the docker file into two stages in this .gitlab-ci.yml :

*   The builder stage (rustdocker/rust..)→ build dependencies and binary
*   The final stage (Alpine) → the build stage

For the CI work i prepared a ready-to-use docker image that contains all i need to make a reliable and fast pipeline for rust project , this image is hosted in my **docker hub .**

> hatembt/rust-ci:latest

This image contains the following packages installed and configured :

*   The `sccache` command : this command caches the compiled dependencies ! so by making this action to our build we can compile deps only one time !! 😅 , and we gained much more time.
*   The `cargo-audit` : it’s a helpful command let’s us to scan dependencies security.

Let’s breakdown the code and understand what’s going on !!

In the first job : **_prepare_deps_for_cargo_** we need our base image **hatembt/rust-ci .**

In this job some setting are required to make a successful build are placed in the **before_script:**

*   Defining the cargo home in the path variable.
*   Defining the cache directory that s generated by `sccache` (it contains the compilation cache ).
*   Adding cargo and rustup ( tey are under .cargo/bin) in the path.
*   Specifying the `RUSTC_WRAPPER` variable in order to use the `sccache` command with the rustc or MUSL in our case.

Now all thing are ready ! so let’s make the build in the **script** section, you are already now what we should do 😃 , let’s skip it 👇.

The **cache and artifacts** sections are very important ! its saves the data under :

*   .cargo/
*   .cache/sccache
*   target/x86_64-unknown-linux-musl/release/material (this is our final binary ).

To know more about caching and artifacts flow this [link](https://gitlab.com/gitlab-org/gitlab-runner/issues/1232).

All data that is created in the first run of the CI jobs will be now saved and uploaded to the Gitlab coordinator. On the next build (new codes are pushed), we will not start the build from scratch, we just build the new packages , the old data will be injected with `<<:*caching_rust` after the **image** keyword.

let’s move on the next JOB : **build_docker_image:**

I made a new Dockerfile for the docker build stage, it’s based on the alpine image and it contain only the **binary** from the previous stage.

The new Dockerfile:

First we need a docker in docker image (dind) → _to get the docker command_ and let’s make the steps below:

*   Login to the Gitlab registry
*   Build the image with the new Dockerfile
*   Push the image to Gitlab registry

and Now the results ! 😧

**_The image size is :_**

![Image for post](https://miro.medium.com/max/60/1*K_OvE1WhvDFMFMETip-Lig.png?q=20)

<noscript><img alt="Image for post" class="t u v du aj" src="https://miro.medium.com/max/2604/1*K_OvE1WhvDFMFMETip-Lig.png" width="1302" height="361" srcSet="https://miro.medium.com/max/552/1*K_OvE1WhvDFMFMETip-Lig.png 276w, https://miro.medium.com/max/1104/1*K_OvE1WhvDFMFMETip-Lig.png 552w, https://miro.medium.com/max/1280/1*K_OvE1WhvDFMFMETip-Lig.png 640w, https://miro.medium.com/max/1400/1*K_OvE1WhvDFMFMETip-Lig.png 700w" sizes="700px"/></noscript>

image sise

**_The CI Time :_**

![Image for post](https://miro.medium.com/max/40/1*mLua8DsSzzvL6nmW5zYLyw.png?q=20)

<noscript><img alt="Image for post" class="t u v du aj" src="https://miro.medium.com/max/778/1*mLua8DsSzzvL6nmW5zYLyw.png" width="389" height="598" srcSet="https://miro.medium.com/max/552/1*mLua8DsSzzvL6nmW5zYLyw.png 276w, https://miro.medium.com/max/778/1*mLua8DsSzzvL6nmW5zYLyw.png 389w" sizes="389px"/></noscript>

> NB: the time is for the hole build time , the build binary and docker_build stages

This is the power of Devops, the art of automation with some **philosophy** in the configurations and the steps to flow we can make even better than these results.

In business the velocity ,the quality and the necessary features (on the application) are very important to Bring the company on the hight levels of success → this is the successful **Digital transformation.**

Finally, i hope that this Story helps you to move on to next steps in the CI/CD systems, you can apply these ideas into any language (mostly complied languages, but still the same steps). If you have any feedback or critiques, please feel free to share them with me. If this walkthrough helped you, please like 👏 the article and connect with me on [LinkedIn](https://www.linkedin.com/in/hatembentayeb/).

# Thank you 😄

