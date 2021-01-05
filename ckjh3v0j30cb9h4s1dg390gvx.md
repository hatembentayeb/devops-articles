## Delivering React .. The hard way !

> In  this Post  we will setup a React pipeline using Gitlab,Ansible and docker. we will go throught the whole process from nothing to a fast, reliable and hightly customizable pipeline with multi-environment deployment. 

<center> <h2>**Daah ! let's start i can't wait	 !**</h2></center>

## Tools

Before we start we need to define the damn   tech stack :

1. **Gitlab** : GitLab is a web-based DevOps lifecycle tool that provides a Git-repository manager providing wiki, issue-tracking and continuous integration and deployment pipeline features.

2. **Ansible** : Ansible is the simplest way to automate apps and IT infrastructure. Application Deployment + Configuration Management + Continuous Delivery.

3. **Docker**: Docker is a tool designed to make it easier to create, deploy, and run applications by using containers.

>*Note: if you dont know nothing about those tools ... No problem .... aaah actually it's a problem  ... we are diving into advanced topic here  ...*


<center><h2></h1>Come on ! i was kidding  </h2></center>
<center><h6> Actually no ... <br>contact me if you need some support</h6></center>

## **Architecture** 

yoo .. we have to draw the global environment architecture  to get the whole picture about what we will do here  ... dont start coding directly. Daaah  ... you have to think by compiling the whole process in mind 

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/vhwnfek050khjp99gcpy.png )

Of course we will create a repository ( i will not explain that ) on gitlab with a hello world react app  ( i will not explain that ) and push it there. 

Let's break down the architecture now : 

* **Block 1** : this where our code application resides and the whole gitlab eco-system also, all configuration to start a pipeline must be there, actually you can install gitlab on your own servers .. but it's not the aim of this post.

* **Block 2**: this is the important block for now (The CI environment)  .. actually it is the server when all the dirty  work resides like buiding docker containers .. saving cache ... testing code and so on ... we must configure this environment with love  haha yeah with love ... it's is the base of the pipeline speed and low level configurations.

* **Block 3** : the target environments where we will deploy our application using ansible playbooks via a secure tunnel .. **SSH** ... BTW i love you SSH  because we will not install any runners on those targets servers we will interact with them only with ansible to ensure a clean deployment.




## **CI environment**

In this section we will connect our gitlab repo to the **CI environment machine** and install the gitlab runner on it of course. 

1. Go to your repo ... under ` settings -->  CI/CD --> runners` and get the gitlab url and the token associeted to ... dont loose it :expressionless:

2. You should have a VPS or a virtual machine on the cloud ... i will work on an azure virtual machine with ubuntu 18.04 installed 
3. Install docker of course ... it's simple [come here](https://docs.docker.com/engine/install/ubuntu/)
4. Installing the gitlab runner :

```bash 
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_<arch>.deb"

dpkg -i gitlab-runner_<arch>.deb
```
Gitlab will be installed as service on your machine but i don't you can encounter a problem when starting it ... (don't ask me i don't know  ) so you can start it as follow : 

```bash 
gitlab runner run & # it will work on background 
```
You can now register the runner with `gitlab-runner register` and follow the instructions ... dont loose the token or reset it ... if you reset the token you have to re-register the runner again. i will make things easier ...  here is my `config.toml` under `/etc/gitlab-runner/config.toml`

```toml
concurrent = 9 
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "runner-name"
  url = "https://gitlab.com/"
  token = "runner-token"
  executor = "docker"
  limit = 0
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    pull_policy = "if-not-present"
    tls_verify = false
    image = "alpine"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache:/cache"]
    shm_size = 0
```

let's make a breakdown here ... 

This runner will run 9 concurent jobs on a docker containers (docker in docker)  based on the alpine container (*to make a clean build*) ... The runner will pull new versions of images if they are not present ... This is optional you can turn it to **always** but we need to speed up the build ... No need to pull the same image again and again if there is no updates ... The runner will save the cache  on the current machine under `/cache` on the host and pass it in use as a docker volume  to save some minutes when gitlab by default upload the zipped cache to it's own storage and download it again ... It's painfull when the cache is becoming huge. At some point on time the cache will be so big .. So you can make your hand dirty and delete the shit 


<center><h2>**We are almost done !**<h2/></center> 

Now you can go the repository under ` settings -->  CI/CD --> runners`  and verify that the runner was registred successfully ( *the green icon* ) 



<center><h2>**. . .**<h2/></center> 


## **The react pipeline** 
let's code the pipeline now  .... wait a second !!! we need the architecture as the previous section ... so here is how the pipeline will look like ... 

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/sxnd34ckr2arhymhm0xv.png)

This pipeline aims to support the folowing features : 
- Caching node modules for faster build
- Docker for shiping containers
- Gitlab private registry linked to the repo 
- Ship only `/build` on the container with nginx web server
- Tag containers with the git SHA-COMMIT
- Deploy containers with an ansible playbook
- SSH configuration as a gitlab secret to secure the target IP
- Only ssh keypairs used for authentication with the target server ... no damn passwords  ... 

<center><h2>**. . .**<h2/></center> 

## **Defining Secrets**
This pipeline needs some variables to be placed in gitlab as secrets on `settings --> CI/CD --> Variables` : 

|Variable name | Role| Type|
|:-:|:-:|:-:|
|ANSIBLE_KEY|The target server ssh private key |file|
|GITLAB_REGISTRY_PASS|Gitlab registry password (your account password )|variable|
|GITLAB_REGISTRY_USER|Gitlab registry login  (your account user )|variable|
|SSH_CFG|The regular ssh config that contains the target IP|file|

The `SSH_CFG` looks like this : 

```ssh 
Host *
   StrictHostKeyChecking no
   
Host dev 
	HostName <IP>
	IdentityFile ./keys/keyfile
	User root

Host staging 
	HostName <IP>
	IdentityFile ./keys/keyfile
	User root

Host prod 
	HostName <IP>
	IdentityFile ./keys/keyfile
	User root
```
I will not explain this  ... [come here](https://linuxize.com/post/using-the-ssh-config-file/)
<center><h2>**. . .**<h2/></center> 

<center><h2>**[KNOCK KNOCK](https://bongo.cat/) ... are you still here **<h2/></center> 

<center>Thank god  ! his here  ... let's continue  then be ready   ... </center> 
<center><h2>**. . .**<h2/></center> 

## **Preparing Dockerfile**

Before writing the `dockerfile` take in mind that the steup should be compatible with the pipeline architecture ... if you remember we have a separate jobs for :
* Installing node modules 
* Run the build process 

So the Dockerfile must contain only the builded assets only to be served by nginx 

Here is our sweet  Dockerfile :
```dockerfile
FROM nginx:1.16.0-alpine
COPY build/  /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d
RUN mv  /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.old
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
This dockerfile does not do too much work, it just take the `/build directory` and copy it under ` /usr/share/nginx/html` to be served.

Also we need a basic nginx config like follow to be under `/etc/nginx/conf.d`: 

```nginx
server {
  include mime.types;
  listen 80;
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
  error_page   500 502 503 504  /50x.html;
  location = /50x.html {
    root   /usr/share/nginx/html;
  }
}
```

You see !  its simple let's proceed to setup the `ansible playbook` for the deployment process ... hurry up
<center><h2>**. . .**<h2/></center> 

## **Deployment with ansible**

We are almost done ! the task now is to write the ansible playbook that will do the folowing :
* Create a docker network and specify the the gateway address
* Authenticate the gitlab registry 
* Start the container with the suitable configurations 
* Clean the unsed containers and volumes 
* Most setup will be in the `inventory file`

Let's take a look at the `inventory_file`: 

```toml
[dev]
devserver ansible_ssh_host=dev ansible_ssh_user=root ansible_python_interpreter=/usr/bin/python

[dev:vars]
c_name={{ lookup('env','CI_PROJECT_NAME') }}-dev #container name
h_name={{ lookup('env','CI_PROJECT_NAME') }}-dev #host name
subnet=172.30.0  # network gateway                                         
network_name=project_name_dev
registry_url={{ lookup('env','CI_REGISTRY') }}                          
registry_user={{ lookup('env','GITLAB_REGISTRY_USER') }}    
registry_password={{ lookup('env','GITLAB_REGISTRY_PASS') }}  
image_name={{ lookup('env','CI_REGISTRY_IMAGE') }}:{{ lookup('env','CI_COMMIT_SHORT_SHA') }}-dev 

[project_network:children]
dev
[project_clean:children]
dev
```
The `ansible_ssh_host=dev` refers to the `SSH_CFG` configuration.

Gitlab by default exports many useful environment variables like :

* `CI_PROJECT_NAME` : the repo name 
* `CI_COMMIT_SHORT_SHA` : the sha commit ID to tag the container

You can explore all variables [here](https://docs.gitlab.com/ee/ci/variables/).

Let's move now to the playbook ... i'm tired damn it haha .. it is a long post ... okay nevermind come on .. 

Here is the ansible playbook : 

```yaml 
---
- hosts: project_network
  #become: yes # for previlged user
  #become_method: sudo   # for previlged user
  tasks:                                                     
  - name: Create docker network
    docker_network:
      name: "{{ network_name }}"
      ipam_config:
        - subnet: "{{ subnet }}.0/16"
          gateway: "{{ subnet }}.1"

- hosts: dev
  gather_facts: no
  #become: yes # for previlged user
  #become_method: sudo   # for previlged user
  tasks:

  - name: Log into gitlab registry and force re-authorization
    docker_login:
      registry: "{{ registry_url }}"
      username: "{{ registry_user }}"
      password: "{{ registry_password }}"
      reauthorize: yes

  - name : start the container
    docker_container:
      name: "{{ c_name }}"
      image : "{{ image_name }}"
      pull: yes
      restart_policy: always
      hostname: "{{ h_name }}"
      # volumes:
      #   - /some/path:/some/path
      exposed_ports:
        - "80"
      networks:
        - name: "{{ network_name }}"
          ipv4_address: "{{ subnet }}.2"
      purge_networks: yes
      
- hosts : project_clean
  #become: yes # for previlged user
  #become_method: sudo   # for previlged user
  gather_facts : no 
  tasks: 

  - name: Removing exited containers
    shell: docker ps -a -q -f status=exited | xargs --no-run-if-empty docker rm --volumes
  - name: Removing untagged images
    shell: docker images | awk '/^<none>/ { print $3 }' | xargs --no-run-if-empty docker rmi -f
  - name: Removing volume directories
    shell: docker volume ls -q --filter="dangling=true" | xargs --no-run-if-empty docker volume rm
```
This playbook is a life saver because we configure the container automatically before starting it ... no setup on the remote host ... we can deploy the same in any other servers based on linux. the container update is quite simple .. ansible will take care of stopping the container and starting new one with different tag and then clean up the shit 

We can also make a `rollback` to the previous container by going to the previous pipeline history on gitlab and restart the lastest job `the deploy job` because we have already an existing container on the registry 

The setup is for `dev` environment you can copy paste the two files for the `prod` & `staging` environment ... 
<center><h2>**. . .**<h2/></center> 

## **Setting up the Pipeline**

The pipeline will deploy to the three environments as i mentioned on the top of this post ... 

Here is the full pipeline code : 

```yaml

variables: 
  DOCKER_IMAGE_PRODUCTION : $CI_REGISTRY_IMAGE 
  DOCKER_IMAGE_TEST : $CI_REGISTRY_IMAGE   
  DOCKER_IMAGE_DEV : $CI_REGISTRY_IMAGE
  

#caching node_modules folder for later use  
.example_cache: &example_cache
  cache:
    paths:
      - node_modules/


stages :
  - prep
  - build_dev
  - push_registry_dev
  - deploy_dev
  - build_test
  - push_registry_test
  - deploy_test
  - build_production
  - push_registry_production
  - deploy_production


########################################################
##                                                                                                                                 ##
##     Development: autorun after a push/merge                                               ## 
##                                                                                                                                 ##
########################################################

install_dependencies:
  image: node:12.2.0-alpine
  stage: prep
  <<: *example_cache
  script:
    - npm ci --log-level=error 

  artifacts:
    paths:
      - node_modules/  
  tags :
    -  runner_name 
  only:
    refs:
      - prod_branch
      - staging_branch
      - dev_branch
    changes :
      - "*.json"

build_react_dev:
  image: node:12.2.0-alpine
  stage: build_dev
  <<: *example_cache
  variables:
    CI : "false"
  script:
    - cat .env.dev > .env
    - npm run build

  artifacts:
    paths:
      - build/
  tags :
    -  runner_name 
  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "dev_branch"'


build_image_dev:
  stage: push_registry_dev
  image : docker:19
  services:
    - docker:19-dind
  variables: 
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  before_script:
  # docker login asks for the password to be passed through stdin for security
  # we use $CI_JOB_TOKEN here which is a special token provided by GitLab
    - echo -n $CI_JOB_TOKEN | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
  script:
    - docker build  --tag $DOCKER_IMAGE_DEV:$CI_COMMIT_SHORT_SHA-dev .
    - docker push $DOCKER_IMAGE_DEV:$CI_COMMIT_SHORT_SHA-dev
  tags :
    - runner_name
  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "dev_branch"'


deploy_dev:
  stage: deploy_dev
  image: willhallonline/ansible:latest
  script:
    - cat ${SSH_CFG} > "$CI_PROJECT_DIR/ssh.cfg"               
    - mkdir -p "$CI_PROJECT_DIR/keys"                            
    - cat ${ANSIBLE_KEY} > "$CI_PROJECT_DIR/keys/keyfile"      
    - chmod og-rwx "$CI_PROJECT_DIR/keys/keyfile"    
    - cd $CI_PROJECT_DIR && ansible-playbook  -i deployment/inventory_dev --ssh-extra-args="-F $CI_PROJECT_DIR/ssh.cfg -o ControlMaster=auto -o ControlPersist=30m" deployment/deploy_container_dev.yml
  after_script:
    - rm -r "$CI_PROJECT_DIR/keys" || true                              
    - rm "$CI_PROJECT_DIR/ssh.cfg" || true

  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "branch_dev"'
  tags :
    - runner_name

########################################################
##                                                                                                                                 ##
##     pre-production: autorun after a push/merge                                            ## 
##                                                                                                                                 ##
########################################################

build_react_test:
  image: node:12.2.0-alpine
  stage: build_test
  <<: *example_cache
  variables:
    CI : "false"
  script:
    - cat .env.test > .env
    - npm run build

  artifacts:
    paths:
      - build/
  tags :
    -  runner_name 

  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "staging_branch"'


build_image_test:
  stage: push_registry_test
  image : docker:19
  services:
    - docker:19-dind
  variables: 
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  before_script:
  # docker login asks for the password to be passed through stdin for security
  # we use $CI_JOB_TOKEN here which is a special token provided by GitLab
    - echo -n $CI_JOB_TOKEN | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
  script:
    - docker build  --tag $DOCKER_IMAGE_TEST:$CI_COMMIT_SHORT_SHA-test .
    - docker push $DOCKER_IMAGE_TEST:$CI_COMMIT_SHORT_SHA-test

  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger" && $CI_COMMIT_BRANCH == "staging_branch"'
  tags :
    - runner_name



deploy_test:
  stage: deploy_test
  image: willhallonline/ansible:latest
  script:
    - cat ${SSH_CFG} > "$CI_PROJECT_DIR/ssh.cfg"               
    - mkdir -p "$CI_PROJECT_DIR/keys"                            
    - cat ${ANSIBLE_KEY} > "$CI_PROJECT_DIR/keys/keyfile"      
    - chmod og-rwx "$CI_PROJECT_DIR/keys/keyfile"    
    - cd $CI_PROJECT_DIR && ansible-playbook  -i deployment/inventory_test --ssh-extra-args="-F $CI_PROJECT_DIR/ssh.cfg -o ControlMaster=auto -o ControlPersist=30m" deployment/deploy_container_test.yml
  after_script:
    - rm -r "$CI_PROJECT_DIR/keys" || true                              
    - rm "$CI_PROJECT_DIR/ssh.cfg" || true
  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger" && $CI_COMMIT_BRANCH == "staging_branch"'
  tags :
    - runner_name

########################################################
##                                                                                                                                 ##
##     Production: must be deployed manually                                                    ## 
##                                                                                                                                 ##
########################################################

build_react_production:
  image: node:12.2.0-alpine
  stage: build_production
  <<: *example_cache
  variables:
    CI : "false"
  script:
    - cat .env.prod > .env
    - npm run build

  artifacts:
    paths:
      - build/
  tags :
    -  runner_name 
  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "prod_branch"'
      when: manual

build_image_production:
  stage: push_registry_production
  image : docker:19
  services:
    - docker:19-dind
  variables: 
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  before_script:
  # docker login asks for the password to be passed through stdin for security
  # we use $CI_JOB_TOKEN here which is a special token provided by GitLab
    - echo -n $CI_JOB_TOKEN | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
  script:
    - docker build  --tag $DOCKER_IMAGE_PRODUCTION:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_IMAGE_PRODUCTION:$CI_COMMIT_SHORT_SHA

  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "prod_branch"'
  tags :
    - runner_name
  needs: [build_react_production]



deploy_production:
  stage: deploy_production
  image: willhallonline/ansible:latest
  script:
    - cat ${SSH_CFG} > "$CI_PROJECT_DIR/ssh.cfg"               
    - mkdir -p "$CI_PROJECT_DIR/keys"                            
    - cat ${ANSIBLE_KEY} > "$CI_PROJECT_DIR/keys/keyfile"      
    - chmod og-rwx "$CI_PROJECT_DIR/keys/keyfile"    
    - cd $CI_PROJECT_DIR && ansible-playbook  -i deployment/inventory --ssh-extra-args="-F $CI_PROJECT_DIR/ssh.cfg -o ControlMaster=auto -o ControlPersist=30m" deployment/deploy_container.yml
  after_script:
    - rm -r "$CI_PROJECT_DIR/keys" || true                              
    - rm "$CI_PROJECT_DIR/ssh.cfg" || true
  rules:
    - if: '$CI_PIPELINE_SOURCE != "trigger"  && $CI_COMMIT_BRANCH == "prod_branch"'
  tags :
    - runner_name
  needs: [build_image_production]
```
Here is some notes about this pipeline:

* The pipeline is protected by default to not be started with the trigger token ( Gitlab pipeline trigger)

* The `prep` stage will start if there is any modifications in any json file including the `package.json` file 
* The pipeline jobs runs on docker alpine image (DinD) so we need some variables to connect to the docker host by using ` DOCKER_HOST: tcp://docker:2375/` and `DOCKER_TLS_CERTDIR: ""`

* The production deployment depends on the staging jobs to be succeeded and tested by the testing team. by default no auto deploy to prod ... it's manual !

* I used some files to store application environment variables using `.env.dev` , `env.test` and `.env.prod` you can use what you want !

* Make sure to use a good docker image for the job based images .. for node i always work with `LTS` versions. 

* Create a `deployment` folder to store the ansible playbooks and inventory files.

* Create a `Cron Job` to delete the cache every three months to clean the cache on the `CI environment`.

* On the target server make sure to install `docker`, `nginx`, `certbot` and `docker python package`
<center><h2>**. . .**<h2/></center> 

## **Final thoughts**

You can make this pipeline as template to deliver other kinds of projects like :
* Python
* Rust 
* Node 
* Go

I hope this post was helpful ! thanks for reading it was great to share this with you, if you have any problems in setting this just let me know !

<center><h2>**Thanks !**<h2/></center> 
