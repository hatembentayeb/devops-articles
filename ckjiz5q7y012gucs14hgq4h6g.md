## Deploying a Dockerized Angular App with Github Actions



> In this article we will discover the devops movement step by step, we will talk about the fundamental concepts then we will make a basic pipeline with github actions to deploy an angular 6 app, so let’s go.

# What is devops ?

Devops is used to remove the conflict between the developers team and the operations team to work together. This conflict is removed by adding a set of best practices, rules and tools. The devops workflow is defined with a set of setps :

![Image for post](https://miro.medium.com/max/60/1*EBXc9eJ1YRFLtkNI_djaAw.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/3964/1*EBXc9eJ1YRFLtkNI_djaAw.png" width="1982" height="1020" srcSet="https://miro.medium.com/max/552/1*EBXc9eJ1YRFLtkNI_djaAw.png 276w, https://miro.medium.com/max/1104/1*EBXc9eJ1YRFLtkNI_djaAw.png 552w, https://miro.medium.com/max/1280/1*EBXc9eJ1YRFLtkNI_djaAw.png 640w, https://miro.medium.com/max/1400/1*EBXc9eJ1YRFLtkNI_djaAw.png 700w" sizes="700px"/></noscript>

devops wokflow

## Plan

This is the first step, where the team defines the product goals and phases, also defining deadlines and assigning tasks to every team member, this step is the root of the hole workflow. the team uses many methodology like scrum and agile.

## Code:

After planning, there is the code when the team converts the ideas to code. every task must be coded and merged to the main app, here we use an **SCM** to organize the collaboration to make a clean code and have a full code history to make a rollback in case of failure.

## Build:

After coding we push the code to Github or Gitlab ( SCM ) and we make the build, usually we use docker images for packaging. also we can build the code to be a Linux package like deb , rpm … or even zip files , also there is a set of tests like unit tests and integration tests. this phase is critical !

## Test:

The build was succeeded, no it’s time to deploy the build artifacts to the staging server when we apply a set of manual and automated tests ( UAT ).

## Release:

it’s the final step for the code work, so we make a release and announce a stable version of our code that is fully functional ! also we can tag it with a version number .

## Deploy:

A pre-prod or a production server is the target now, to make our app up and running

## Operate:

It’s all about infrastructure preparation and environment setup with some tools like terraform for IaaC, ansible for configuration management and security stuff configurations …

## Monitor:

The performance is very important, so we install and configure some monitoring tools like ELK, nagios and datadog to get all information about the applications like CPU and memory usage …



# Deploying an angular app

In this example we will deploy a simple angular app on two environments.

*   On VPS ( OVH provider) as a development server.
*   on heroku as a staging server.

So you must have a VPS and a heroku account to continue with me.

The application repository is here : [Github repo](https://github.com/hatembentayeb/angular-devops).

1.  Clone the project with `git clone https://github.com/hatembentayeb/angular-devops`
2.  run `npm install && ng serve` to run the app locally

## Preparing the deployment for heroku

Nginx is a popular and powerful web server can be used to serve a large variety of apps based on python, angular and react …

I will go through an optimization process to produce a clean and a lightweight docker container with the best practices as much as i can.

## Writing the Dockerfile

First we will prepare the Dockerfile to be deployed to the heroku cloud,so there is some tricks to make it work smoothly, make sure that you have an account and simply click new to create an app :

![Image for post](https://miro.medium.com/max/60/1*nti0ILLxeBTzz3GmB8ZeTA.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/1732/1*nti0ILLxeBTzz3GmB8ZeTA.png" width="866" height="411" srcSet="https://miro.medium.com/max/552/1*nti0ILLxeBTzz3GmB8ZeTA.png 276w, https://miro.medium.com/max/1104/1*nti0ILLxeBTzz3GmB8ZeTA.png 552w, https://miro.medium.com/max/1280/1*nti0ILLxeBTzz3GmB8ZeTA.png 640w, https://miro.medium.com/max/1400/1*nti0ILLxeBTzz3GmB8ZeTA.png 700w" sizes="700px"/></noscript>

create heroku app

Make sure to give a valid name for your app, then go to your **accout settings** and get your `API_KEY` that we will use it in the pipeline file:

![Image for post](https://miro.medium.com/max/60/1*bO0u_UUltF_8lXxnynqq9g.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2698/1*bO0u_UUltF_8lXxnynqq9g.png" width="1349" height="505" srcSet="https://miro.medium.com/max/552/1*bO0u_UUltF_8lXxnynqq9g.png 276w, https://miro.medium.com/max/1104/1*bO0u_UUltF_8lXxnynqq9g.png 552w, https://miro.medium.com/max/1280/1*bO0u_UUltF_8lXxnynqq9g.png 640w, https://miro.medium.com/max/1400/1*bO0u_UUltF_8lXxnynqq9g.png 700w" sizes="700px"/></noscript>

getting api key

let’s take a look at the `dockerfile` of the app:

This Dockerfile is splitted into two stages :

*   **Builder stage :** The name of the stage is builder, it is a temporary docker container that produces an artifact which is the `dist/` folder created by `ng build --prod` that compiles our project to produce a single `html` page and some `*js & *.css` . The base images that is used here is `trion/ng-cli` that containes all requirements to run an angular up and it’s accessible for public use in the `Docker-hub`, the public docker registry.
    Make sure to install all app requirement packages with `npm ci` , the `ci` command is used often in the continues integration environments because it is faster than `npm install.`
*   **Final stage:** The base image for this stage is `nginx:1.17.5` and simply we copy the `dist/` folder from the `builder` stage to the `/var/share/nginx/html` folder in the nginx container with the command `COPY --from=builder ...`
    There is additional configurations required to run the app, we need to configure nginx, there is a file named `default.conf.template` that contains a basic nginx configurations so we copy it to the container under `/etc/nginx/conf.d/default.conf.template` , this file have the **_$PORT_** variable that have to be changed when building the docker image in the heroku environment.
    The `default.conf.template` :


```
server {                         
listen $PORT default_server;

location / {                           
include  /etc/nginx/mime.types;                                                      root   /usr/share/nginx/html/;
index  index.html index.htm;       
}                                                                       }</span>
```


Also make sure to copy the `nginx.conf` under the `/etc/nginx/nginx.conf` , you are free to change and modify 😃, but for now i will use the default settings.
The last command is a little bit confusing so let’s break it down :


```
CMD /bin/bash -c “envsubst ‘\$PORT’ < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf” && nginx -g ‘daemon off;’</span>
```


→ **_/bin/bash -c ‘ command’ :_** This command will run a linux command with _the bash shell.
→_ **_envsubst_** : It is a program substitutes the values of environment variables, so it will replace the **$PORT** from the heroku environment and replace it in the `default.conf.template` file with it’s value, this variable is given by heroku and attached to your app name, then we rename the template with `default.conf` which is recognized by nginx.
→ **_nginx -g ‘daemon off;’_** : The `daemon off;` directive tells Nginx to stay in the foreground. For containers this is useful as best practice is for one container = one process. One server (container) has only one service.

## Preparing the deployment for the VPS on OVH

We will use the VPS as a development server so no need for a docker now we will use ssh for this, after all make sure to have a VPS , ssh credentials and a public IP.

I assume you have `nginx` installed , if not try to do it, it is simple 😙

In this tutorial i will be using the **_sshpass_** command, it is powerful and suitable for CI environments.
You can install it with : `apt-get install sshpass -y .`

lets deploy the app to our server from the local machine, navigate to the repo and run `ng build --prod` , then navigate to `dist/my-first-app` folder and type this command :


```
sshpass  scp -v -p <password>  -o stricthostkeychecking=no -r *.* root@<vps-ip>:/usr/share/nginx/html</span>
```


If you don’t want to hardcode the password in the command line try to set the `SSHPASS` variable with your password like this `export SSHPASS="password"` and replace `-p` with `-e` to use the environment variable.

Now all things almost done ! great 😃 ! let’s prepare the **pipeline** in the github actions which is a fast and powerful ci system provided by github inc.

Under the project root folder create the file `main.yml` in the `github/wokflows` folder, this directory is hidden so must start with a point like this : `.github/workflows/main.yml`

## Preparing the pipeline

let’s take a look at the pipeline steps and configurations :

.github/workflows/main.yml

*   **Block 1**: In this block we define the the workflow name and the actions that must be performed to start the build , test and the deployment. and of course you have to specify the branch of your repo (by default `master` ).
*   **Block 2** : The `jobs` keyword has to sub keywords `build` and `steps` , the build define the base os for the continues integration environment, in this case we will use `ubuntu-latest` , also we define the `node-version` as a matrix that allow us to use multiple node versions in the list, in this case we need only `12.x` . The steps allow us to define the wokflow steps and configurations ( build,test,deploy...).
*   **Block 3** : `actions/checkout@v1` is used to clone the app code in the ci env. this action is provided by github.
    Lets define a `cache` action with the name `cache node modules` , the name is up to you 😃, then we use a predefined action called `actions/cache@v1` and specify the folders that we want to cache.
*   **Block 4** : Installing and configuring the node run-time with an action called `actions/node-setup@v1` and pass to it the desired node version that we already defined.
*   **Block 5** : The show will begin now ! let’s configure the build and the deployment to the VPS. Create two environment variables `SSHPASS` for the sshpass command and define the `server` address , make sure to define these values on the github secrets under setting on the top of your repo files. Under `run` keyword put your deployment logic. so we need the sshpass command and and the angular cli to be installed, then install all required packages and build the app with the production mode `--prod` , next, navigate to the `dist/my-first-app` folder and run the sshpass command with a set of arguments to remove older app in the server and deploy the new code.
*   **Block 6** : Now heroku is our target, so define also two env. variables, the heroku registry url and the API KEY to gain access to the registry using docker , next we need to define a special variable `HEROKU_API_KEY` that is used by heroku cli, next, we login to the heroku container and build the docker image then we pushed to the registry. we need to specify the target app in my case i named it `angulardevops` . After deploying the docker image we need to release it and tell the heroku dynos to run our app on a heroku server, using 1 server `web=1` , note that `web` is the name of the docker image that we already pushed.

We are almost done! now try to make a change in the app code and push it to GitHub , the workflow will start automatically 🎉 🎉 😄 !

![Image for post](https://miro.medium.com/max/60/1*O2HJGDQaLjiL4ePKNdHHWw.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2732/1*O2HJGDQaLjiL4ePKNdHHWw.png" width="1366" height="620" srcSet="https://miro.medium.com/max/552/1*O2HJGDQaLjiL4ePKNdHHWw.png 276w, https://miro.medium.com/max/1104/1*O2HJGDQaLjiL4ePKNdHHWw.png 552w, https://miro.medium.com/max/1280/1*O2HJGDQaLjiL4ePKNdHHWw.png 640w, https://miro.medium.com/max/1400/1*O2HJGDQaLjiL4ePKNdHHWw.png 700w" sizes="700px"/></noscript>

GitHub actions ci env

You can view the app here: [https://angulardevops.herokuapp.com/](https://angulardevops.herokuapp.com/)

![Image for post](https://miro.medium.com/max/60/1*P_cruJ70nMbv363xAy8cKg.png?q=20)

<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/2732/1*P_cruJ70nMbv363xAy8cKg.png" width="1366" height="768" srcSet="https://miro.medium.com/max/552/1*P_cruJ70nMbv363xAy8cKg.png 276w, https://miro.medium.com/max/1104/1*P_cruJ70nMbv363xAy8cKg.png 552w, https://miro.medium.com/max/1280/1*P_cruJ70nMbv363xAy8cKg.png 640w, https://miro.medium.com/max/1400/1*P_cruJ70nMbv363xAy8cKg.png 700w" sizes="700px"/></noscript>

heroku app deployment

Finally, this tutorial is aimed to help developers and DevOps engineers to deploy an Angular app, I hope it is helpful 😍. for any feedback please contact me!

_If this post was helpful click the clap button as much as possible 😃._

# Thank you 😃

