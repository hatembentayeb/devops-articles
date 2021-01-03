## Automate you Documentation with Gitlab and Mkdocs

<span class="s"></span>![Image for post](https://miro.medium.com/max/60/1*kAsK8EV8ukMDnrwnsTHBHQ.png?q=20)


<noscript><img alt="Image for post" class="t u v ek aj" src="https://miro.medium.com/max/3840/1*kAsK8EV8ukMDnrwnsTHBHQ.png" width="1920" height="960" srcSet="https://miro.medium.com/max/552/1*kAsK8EV8ukMDnrwnsTHBHQ.png 276w, https://miro.medium.com/max/1104/1*kAsK8EV8ukMDnrwnsTHBHQ.png 552w, https://miro.medium.com/max/1280/1*kAsK8EV8ukMDnrwnsTHBHQ.png 640w, https://miro.medium.com/max/1456/1*kAsK8EV8ukMDnrwnsTHBHQ.png 728w, https://miro.medium.com/max/1632/1*kAsK8EV8ukMDnrwnsTHBHQ.png 816w, https://miro.medium.com/max/1808/1*kAsK8EV8ukMDnrwnsTHBHQ.png 904w, https://miro.medium.com/max/1984/1*kAsK8EV8ukMDnrwnsTHBHQ.png 992w, https://miro.medium.com/max/2000/1*kAsK8EV8ukMDnrwnsTHBHQ.png 1000w" sizes="1000px"/></noscript>

# Automate your Documentation with Gitlab and Mkdocs


> Producing documentation may be painful and need a lot of time to write and operate. In this story, i will share with you, my way of generating docs using the devops approach. To make life easier, we will explore the art of automation 😃.

> Let’s go folks 😙

# Create a Gitlab repo

This is straightforward, follow these steps:

*   Log in to your GitLab
*   Click new project
*   Give it a name: `auto_docs`
*   Initialize it with a `README.md` file
*   Make it public or private
*   Hit create

Now clone the project by copying the URL and run this command :


```
$ git clone [https://gitlab.com/auto_docs.git](https://gitlab.com/auto_docs.git)</span>
```


# Setting up the environment

I’m using a Linux environment but it is possible to reproduce the same steps on a Windows machine.

In order to follow me, you need a set of tools that must be available on your machine … make sure to to have `python3` installed, I have python 3.8 (latest).

## Creating a virtual environment

The easiest way to set up a virtual environment is to install `virtualenv` python package by executing `pip install virtualenv` .

Navigate to your local GitLab repository and create a new virtual environment.


```
$ cd auto_docs/
$ virtualenv autodocs
$ source autodocs/bin/acivate</span>
```


## Installing Mkdocs Material

Make sure that the virtual environment is active.

Install the mkdocs material with this command : `pip install mkdocs-material.`
This package needs some dependencies to work .. install them by using a requirement.txt file, copy-paste the dependencies list to filename `requirements.txt`


```
Babel==2.8.0
click==7.1.1
future==0.18.2
gitdb==4.0.4
GitPython==3.1.1
htmlmin==0.1.12
Jinja2==2.11.2
joblib==0.14.1
jsmin==2.2.2
livereload==2.6.1
lunr==0.5.6
Markdown==3.2.1
MarkupSafe==1.1.1
mkdocs==1.1
mkdocs-awesome-pages-plugin==2.2.1
mkdocs-git-revision-date-localized-plugin==0.5.0
mkdocs-material==5.1.1
mkdocs-material-extensions==1.0b1
mkdocs-minify-plugin==0.3.0
nltk==3.5
Pygments==2.6.1
pymdown-extensions==7.0
pytz==2019.3
PyYAML==5.3.1
regex==2020.4.4
six==1.14.0
smmap==3.0.2
tornado==6.0.4
tqdm==4.45.0</span>
```


Install them all with one command : `pip install -r requirements.txt`
Now it’s time to create a new mkdocs project 😅.

Run this command : `mkdocs new .` and verify that you have this structure :


```
|--auto_docs
    |--- docs
    |--- mkdocs.yml</span>
```


*   The **docs** folder contains the structure of your documentation, it contains subfolders and markdown files.
*   The **mkdocs.yml** file defines the configuration of the generated site.

Let's test the installation by running this command : `mkdocs serve` . The site will be accessible on [http://locahost:8000](http://locahost:8000) and you should see the initial look of the docs.

# Setting up the CI/CD

let’s enable le CI/CD to automate the build and the deployment of the docs. Notice that GitLab offers a feature called `gitlab pages` that can serve for free a static resource (HTML, js, CSS). The repo path is converted to an URL to your docs.

## Create the CI/CD file

Gitlab uses a YAML file — it holds the pipeline configuration.

The CI file content:


```
stages :
  - build</span><span id="3031" class="fu lb ji ex kt b lc mb mc md me mf le s lf">pages:
  stage: build
  image:
  name: squidfunk/mkdocs-material
  entrypoint:
    - ""
  script:
    - mkdocs build
    - mv site public
  artifacts:
    paths:
      - public
  only:
    - master
  tags:
    - gitlab-org-docker</span>
```


This pipeline uses a docker executor with an image that contains `mkdocs` already installed… mkdocs build the project and put the build assets on a folder called `site` … to be able to use GitLab pages you have to name your job `pages` and put the site assets into a new folder called `public.`

For tags: check the runner's section under **settings → CI/CD →Runners** and pick one of the shared runners that have a tag _GitLab-org-docker._

All things were done 🎉 🎉 😸 !

Oh ! just one thing … we forgot the virtual environment files .. they are big and not needed on the pipeline … they are for the local development only. The mkdocs image on the pipeline is already shipped with the necessary packages.

So … create a new file called `.gitignore` and add these lines:


```
auto_docs/ 
requirements.txt</span>
```


The **auto_docs** folder has the same name as the virtual environment .. don't forget 😠! you will be punished by pushing +100Mi 😝 and you will wait your whole life to complete the process haha 😢.

Now run `git add . && git commit -m "initial commit" a && git push` … go to your GitLab repo and click **CI/CD → pipelines,** click on the blue icon and visualize the logs .. once the job succeeded, navigate to **settings -> pages** and click the link of your new documentation site (you have to wait for 10m~ to be accessible)

> Finally, I hope this was helpful ! thanks for reading 😺 😍!

![Image for post](https://miro.medium.com/max/60/0*Piks8Tu6xUYpF4DU?q=20)

