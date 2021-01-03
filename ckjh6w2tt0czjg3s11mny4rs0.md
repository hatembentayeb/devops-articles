## Gitlab and DevSecOps Pipeline Implementation — Part I

<span class="s"></span>![Image for post](https://miro.medium.com/max/60/1*bFgG2vbvkoHki-M_pT6qng.png?q=20)


<noscript><img alt="Image for post" class="t u v dp aj" src="https://miro.medium.com/max/3200/1*bFgG2vbvkoHki-M_pT6qng.png" width="1600" height="900" srcSet="https://miro.medium.com/max/552/1*bFgG2vbvkoHki-M_pT6qng.png 276w, https://miro.medium.com/max/1104/1*bFgG2vbvkoHki-M_pT6qng.png 552w, https://miro.medium.com/max/1280/1*bFgG2vbvkoHki-M_pT6qng.png 640w, https://miro.medium.com/max/1456/1*bFgG2vbvkoHki-M_pT6qng.png 728w, https://miro.medium.com/max/1632/1*bFgG2vbvkoHki-M_pT6qng.png 816w, https://miro.medium.com/max/1808/1*bFgG2vbvkoHki-M_pT6qng.png 904w, https://miro.medium.com/max/1984/1*bFgG2vbvkoHki-M_pT6qng.png 992w, https://miro.medium.com/max/2160/1*bFgG2vbvkoHki-M_pT6qng.png 1080w, https://miro.medium.com/max/2700/1*bFgG2vbvkoHki-M_pT6qng.png 1350w, https://miro.medium.com/max/3200/1*bFgG2vbvkoHki-M_pT6qng.png 1600w" sizes="1600px"/></noscript>

Devsecops image [source](http://ictvietnam.vn/files/tccntt/source_files/2019/07/10/190710-de-202150-100719-76.jpg)

# Gitlab and DevSecOps Pipeline Implementation — Part I


> As a **devops** engineer you may want to improve your pipeline security to deliver more reliable and qualified software to the market and gain the client trust.
> In this story i will go through the security process in the traditional **devops** pipelines.I assume all readers know about devops and all related stuff like pipelines,docker … .

# **Why Gitlab ?**

Gitlab is an advanced source code management based on GIT and it’s much like Github with new features like unlimited private repositories and a CI/CD configuration for your projects.

In this story we will concentrate on a special file named `.gitlab-ci.yml` which is the main file for the Continuous Integration and Continuous Delivery/Deployment.It is a `YML` file which is very organized and easy to read and understand.

# Defining the Pipeline

This pipeline can be applied to any project, but here we will work with a **nodejs** application that is must be a set of micro services that interact with each other to do the remain job.

## The pipeline variables

There is two ways to define variables in Gitlab:

*   Secret variable : don’t hard-code these variable in your CI file ! there is a special place in gitlab to define them.
*   Other variables : like app names, registry names, urls … u can define them in your CI file. see this example :


```
variables:
    PIPELINE_MODE : medium 
    DOCKER_DRIVER : overlay2
    ACCOUNTS_IMAGE : $CI_REGISTRY_IMAGE/accounts:$CI_COMMIT_SHA
    ...
    GITLAB_URL : gitlab.com</span>
```


## The Pipeline stages :

The **_stages_** keyword:

let’s breakdown the code :

*   **prep:** the preparation stage is used to prepare all the required dependencies to get the apps up and running.
*   **test:** testing the app with a different set of tests including unit tests and static analysis security tests alongside with code quality.
*   **build_images:** using docker to build every service as docker container with the commit ID as tag.
*   **deploy-staging:** run the app in a test server with docker compose.
*   **dast :** apply a dynamic analysis security tests on the staging server
*   **performance :** test the app performance like the css, js and images sizes when loading the app.
*   **clean:** we will use a self hosted Gitlab runners so we have a clean after each build ( removing unused containers …).

## — Prep — Stage:

The preparation step is required in every CI/CD pipeline, it helps to install all packages needed by the app to run. let’s look at the example of a nodejs app :

*   **install_dependencies** : This is the job name .
*   **image** : We use node:10.15.1 as a runtime environment
*   **stage** : Must have a value from the stages names to make the visual preview of the pipeline
*   **<<:*default-cache** : Inject the cache into the current job
*   **script** : Do some work here like installing packages in this case.
*   **artifacts** : Save the result packages w define path, we have to save the node modules folder
*   **tags** : If you have a custom runner on a custom CI environment , connect your gitlab-runner with the tag to make this job on your CI virtual machine .
*   **only** : If you want to use a specific branch ( put the name in `refs` tag), by default it uses the master branch .
*   **variables** : i used a **muli-mode** pipeline so if you define the `PIPELINE_MODE = medium` will run all jobs that contains this variable ( run a job after validating the variable content).

Every **install dependencies** must have a cache declaration to save all downloaded stuff to be used later in the next builds, this step can save you a lot of time and make the pipeline faster. you know node modules it’s much like the hell 😠

let’s implement the **cache** step :

## — Test — stage :

lets name it the validation stage 😃 because it is the bridge that defines the quality of our code in term of security, structure and unit testing so in this step we will define three jobs , let’s see the example bellow :

let’s begin with the first job : **dependencies-security** , we a special command in the **script** tag : the `npm audit && echo "ok" || echo "not OK"` , the audit sub-command is available only in the **npm** version 6 and above. it can analyze all dependencies and search for potential security issues by consulting the **CVE database.** This command`... && echo "ok" || echo "not OK"` is used to prevent the pipeline crash, just print the log of the command and continue , you can also export a report as a **json** file and pass it as an artifact.

This job runs only on the `medium` and `advanced` mode of our pipeline.

**unit-test :** before running this stage make sure to provide a directory called tests/test that contains all needed tests, you can use `mocha` `karma` …
they are frameworks to write your unit tests.to run the test just call `npm run test` that’s all 😃.

**codequality :** this stage is very important ! and has a direct contact with the developers team, so it checks the code quality by like how much code must put in a single function , in a single class , it detect replicated code blocks and much more, we use to check javascript code, css code and much more plugins for others languages.

let’s break down the code ( you already know some of them), we will use docker image with **DIND** (docker in docker → docker cli command),let’s look at the script tag :

*   Create a directory named `public` it’s a special one to view a static html file in Gitlab.
*   Pull the **_codeclimate_** docker image which is a _code analyzer tool._
*   Run the image and pass some parameters :
    → `_--name_` _give the container a name to use it later
    →_ `--env CODECLIMATE_DEBUG=1` : tell codeclimate to make an expanded output .
    → `--env CODECLIMATE_CODE="$PWD"` : specify the code path , in this case we use `pwd` : current directory.
    → `--volume "$PWD":/code` : make a volume to map teh working directory to `/code` in the container.
    → `--volume /var/run/docker.sock:/var/run/docker.sock` : make a volume to map the docker socket, it’s used to communicate with the docker API (the docker daemon).
    → `—volume /tmp/cc:/tmp/cc` : the volumes is used to persist data generated in the docker container in the host ( CI env).
    → `codeclimate/codeclimate:0.85.4` : this is the image name to perform the analyze operation with the command `analyze -f codeclimate.html` , the result will be a html file generated by a redirection `> codeclimate.html` .
    → now we need to use the `app review` feature of gitlab to view the htmll file directly from gitlab so move the html file to the `public` directory.

## — Container Security — Stage

When you work with docker containers you should be careful to security issues when using base images like ubuntu, alpine and centos …

The docker image is a set of layers some of them are locked and the others no , the base system image are locked and the layer you add it ( your code ) is not, so before using the image try to analyze it or try to build a reliable and optimized image, that may help you a lot to make new images based on your previous image.

You can add a stage in your pipeline to test on docker security containers and often the major security issues are in the system packages that might be out dated or unpatched. so here is an example of docker container security integration in gitlab :

To get access to the gitlab registry, you must login with docker with a special trick provided by gitlab.
The idea is : you are already logged in with your account so no need to login again; so gitlab provide a user called `gitlab-ci-tocken` and a secret token stored in a variable called `CI_JOB_TOKEN` are generated in the job scope to make the login easier and much easy.

let’s breakdown the code :

1.  Build a dev docker image
2.  Push it to the gitlab registry
3.  Install needed packages bash and curl
4.  Install `anchore` the container security tool and pass the image name to it, it will produce a full report in different json files containing a lot of information about your image.
5.  Using the powerful gitlab API, download the `anchore_script.sh` to filter the json files and print all infected packages.


```
#!/bin/bash 
      for f in anchore-reports/*; do
        if [[ "$f" =~ "content-os" ]]; then
          printf "\n%s\n" "Installed packages on ${IMAGE_NAME}:"
          jq '[.content | sort_by(.package) | .[] | {package: .package, version: .version}]' $f || true
        fi
        if [[ "$f" =~ "vuln" ]]; then
          printf "\n%s\n" "List vulnerabilities on ${IMAGE_NAME}:"
          jq '[.vulnerabilities | group_by(.package) | .[] | {package: .[0].package, vuln: [.[].vuln]}]' $f || true
        fi
      done</span>
```


I will not explain the script 😃 just test it and figure out his purpose 😄

**Finally**, i hope the article is helpful, there is a Part II **_SOON_** to complete the rest of stages. if you have any feedback please contact me.

# Thank you 😃

