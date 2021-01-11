## Hello Docker 😃 — Part II



> Hello to the second part of the “_hello docker “_ series, if your are new on docker please check the previous part by following this [_hello docker 😃 — Part I_](https://hatembentayeb.hashnode.dev/hello-docker-part-i)_,_ In this lecture i will show you more advanced feature of the docker command line, also we will create a basic dockerized project for a simple python app with flask and we will push to [dockerhub](https://hub.docker.com/).

# Common Docker commands

In this section I am going to show you the most used docker commands so let’s begin :

* To get info about your docker environment use : `docker info` 
* To remove a container use : `docker rm <name | ID>`
* To remove an entire image use : `docker rmi <name | ID>`
* To remove a container after run use : `docker run --rm <name|ID>` 
* To view current running containers use : `docker ps`
* To view all running and exited containers use: `docker ps -a`
* To view all and only the IDs of containers use : `docker ps -a -q` 
* To get all images use : `docker images` 
* To remove all images use : `docker rm $(docker ps -a -q)`
* To view the _<none>_ images use : `docker images -f"dangling=true"`
* To remove them use : `docker rmi $(docker images -f"dangling=true" -q)` 
* To run the container in background use : `docker run -d ... <name|ID>`

That’s enough for now 😆, no lets build a simple python project with flask

# The Flask app


Flask is a simple and powerful Framework for python like apache or tomcat... So let's begin :

The `app.py` file
```python 

from flask import Flask
app = Flask(__name__)

@app.route("/")
    def hello():
        return "Hello My Name is Hatem"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```
Make sure to install `flask` with `pip install flask` and run the script with `python app.py` and open your browser on `localhost:8080/` .

Let’s generate the `requirements.txt` , we will not use `pip freeze` here (we are not in a virtual environment) but we will use `pipreqs` and make sure to install it with `pip install pipreqs` .

use `pipreqs <path to the python project>` to generate it.

Now let’s write the Dockerfile :

Dockerfile

```yaml
FROM python:latest
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD python app.py
```

* `FROM python:latest` : using python as a base image.
* `WORKDIR /app` : using _/app_ as a default directory.
* `COPY requirements.txt .` compy this file to _/app_ `RUN pip install -r req..txt` : install the required dependencies
* `COPY app.py .` : copy the app script to _/app_ 
* `CMD python app.py` : The container entry point when we run it
* Run the build process with `docker build -t hello_flask .`

Run the container `docker rn --rm -p 8080: 8080 hello_flask` , the `-p 8080:8080` : _will map the port_ _8080 on your host to the port 8080 in your container → this is the port mapping in docker, you can change the host port to any port you want and must be > 1024 ( ports < 1024 are reserved by the system)._

# Environment variables in Docker

An [_environment variable_](https://en.wikipedia.org/wiki/Environment_variable) is a variable whose value is set outside the program, typically through the functionality built into the operating system or microservice. An environment variable is made up of a name/value pair, and any number may be created and available for reference at a point in time.
→ [source](/chingu/an-introduction-to-environment-variables-and-how-to-use-them-f602f66d15fa) for more information.

let’s use them in our example :


```python
import os 
@app.route("/")
def hello():
    return "Hello My Name is {}".format(os.environ['NAME'])
```


the `os.environ['NAME']` will fetch the `NAME` variable and get its value and return it in the browser.
make sure to build the container again and run it like this :
`docker run --rm -p 8080:8080 -e "NAME=steve" hello_flask`

# Volumes with Docker

In order to be able to save (persist) data and also to share data between containers, **Docker** came up with the concept of **volumes**. Quite simply, **volumes** are directories (or files) that are outside of the default Union File System and exist as normal directories and files on the host filesystem.

→ [source](https://blog.container-solutions.com/understanding-volumes-docker)

let’s make some change to our script :


```python
@app.route("/name_from_file")
def name_from_file():
    with open("files/name.txt","r") as file :
        name= file.readline()
    return "Hello My Name is {}".format(name)
```


Make sure to add a directory named _“files”_ and add a file named _“name.txt”_ that contains your name or whatever you want.

Now make the build again ! and run it with this command :


```bash
docker run --rm -v ${PWD}/files:/app/files -e "NAME=steve" -p 8080:8080 hello_flask
```


Now open your browser and type `localhost:8080/name_from_file` , you should see the name that you already add it in the _name.txt_ file. In this case, you can change the content of the file in real-time and reload the page, you should see the changes 😆.

# Saving and publishing images

After finalizing your work there are two things you should do :

*   Saving images to tarballs or compressed archives
*   Publishing images to registries to be used in public or private…

Let's begin by saving our example image to a tarball by running this command `docker save --output hello_flask.tar hello_flask` , now, check your current directory and type in the terminal `ls -sh hello_flask.tar` to get the archive size. if you want to reduce the archive size use the `gzip` command which is compression tools. Execute this command :


```bash
docker save hello_flask | gzip > hello_flask.tar.gz
```


and check the size again 😃. you can now upload them to your cloud storage or wherever you want. Now if you want to load the tar file to the docker engine simply run :


```bash
docker image load -i hello_flask.tar
```


Docker hub is our target now, to publish images in it you have to make an account first. the image name must be with this syntax `account_name/image_name:image_tag` so let's rename our image by running this command :


```bash
docker tag hello_flask hatembt/hello_flask:latest
```


Now you have to log in to your account and push the image to the public :


```bash
docker login 
docker push hatembt/hello_flask:latest
```


To pull the image just run `docker pull hatembt/hello_flask:latest` , no need for credentials here 😃 .

**Finally**, I hope that this tutorial is helpful to everyone who wants to know docker. In Part III, we will go through a complex project ( Front-end + Back-end + Database) and we will use docker-compose as an orchestrator. If you have any feedback please share it with me [Hatem Ben Tayeb](https://www.linkedin.com/in/hatembentayeb/) 😆.

# Thank you 😃

