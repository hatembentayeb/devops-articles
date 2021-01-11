## Hiding my nodejs application code within a docker container


> Building binaries on compiled languages like Rust and Go as Static or dynamic linking makes a huge step on code security, I mean no one can access your code, reading it !, but in many use cases they can by doing some complicated reverse engineering methods ... actually it's hard! it becomes harder when you built your app as a static linking and bundle all the shit 💩 together. Nodejs applications can be read it easily ... all your source code is accessible, In this post, we will hide our node app and "**COMPILE**" it! seems awesome right! let's go folks  😇 !

# Get the app 

You need to have nodejs installed to test the app locally, I tested the app on `node 12.2.0`. You might have some problems with the database so make sure to use the appropriate `mongoose` version that is compatible with `MongoDB`, I have mongoose `5.10.10` that is compatible with mongo `4.4`. Let's run the app 🤓 !

### Clone the code 
Make sure you have Git installed!
```
git clone https://github.com/hatembentayeb/node-rest-api-mongo.git 
cd node-rest-api-mongo
npm install
```

### Run a mongo instance
make sure you have docker installed, otherwise install mongo directly.

`docker run -p 27017:27017  --name mymongo -d mongo:latest`

### Modify the .env file
On the mongo uri we have the ip of the mongo container, you can use `localhost:27017` instead, but we need the IP when we run the app inside the container.

```
PORT=3000
MONGODB_URI=mongodb://172.17.0.3:27017
DB_NAME=hatem
```

### Run the app 
You can run it with `node app.js` 

`npm start`

### Insert data 
Using curl with `POST` request to send data inside the `products.json`. the `jq` command is just for formatting JSON.
```
curl  -X POST -H "Content-Type: application/json" "http://localhost:3000/products" -d @products.json | jq 
```

### Get data 
Simple `GET` request to get all requested data

```
curl http://localhost:3000/products | jq
```

# Why we need a binary 


This approach! i mean building binary can save us especially to get rid of the `node_modules` inside our container! we will save space and enhance the image size! besides we don't need any base node image like `FROM node: latest` because it's just a binary that needs a shell to run, also you have to make a choice of your build platform or architecture, you can find the full list of the available architectures for `nexe` on this link [full list](https://github.com/nexe/nexe/releases). 

In this tutorial, we need a minimal docker image ... like  `5MB` of size ! , that's awesome ! yes, it's is the `alpine` image, the smallest one! now we can deploy a docker image with a low size! it will be extremely fast believe me, if you have a small server with limited os storage or an on-premise docker registry with limited storage it will be wonderful to store images with small size  !! 

If you try to build the project with `nexe` and then use it on a docker container based on alpine, it will work !, only in your dreams 🤣. To be able to run it on alpine we need to set the right architecture platform before the build. 

In terms of security, you are almost secure they can't get to your source code! it's a binary! dude, but they can access your container if they succeeded to create shell access with the `RCE (remote code execution) attack` ... they can do it by attacking your application and exploit it,  so they are on your container now! they can't read your code but they can crash your container and access your host and here is the disaster 😳 ! so I will try to limit the privileges on the container by disabling the root access and create a non-root user and remove some of the default Linux commands like `ls, cat and mv` 😈. 

**Anyways, let's start building binaries 😋**

# Let's build it 

In order to build the binary, we need a special package called `nexe` with over then `9k` stars on GitHub 🧐 !!

To install it run this command `npm install -g nexe` and check the package version `nexe -v`, I have the `4.0.0-beta.16` version. 


To build the application just run this command! :

```
next app.js -t alpine-x86-12.2.0 -o mybinary
```

let's breakdown the nexe options : 

* `app.js`: is your application entry point
* `alpine-x86-12.2.0`: is the node version that is compatible with the alpine architecture platform with is `x86` 
* `mybinary`: is your binary output file that we will copy to the container.


In some cases you need to bundle some `HTML` files to your binary or any some directories, you can do that by adding the `--recursive/-r ` option and specify the path like this :

```
nexe app.js -t alpine-x86-12.2.0 -r static/**/*.html -o mybinary
```

Alright let's run the binary : 

```
$ ./mybinary 

Server started on port 3000...
Mongoose connected to db...
Mongodb connected....

```

And here we go! it's much like a command! try to move it to another location on your file system and run it again! awesome it works smoothly, congrats! you have a standalone binary for your node app  😍 !


# Move it to docker

let's run the app with docker, we need a dockerfile to build the image, here is an example: 

```
FROM alpine
WORKDIR app
RUN adduser --disabled-password btx
RUN rm -f /bin/cat /bin/ls /bin/mv /bin/find /bin/cd 
COPY hatem .
COPY .env .
EXPOSE 3000
RUN chown -R btx:btx /app
USER btx
CMD  ["./mybinary"]
```

We will base the container on a lightweight docker image: **Alpine** it's about 5MB in size! then we define our working directory **/app** to be the default directory for the rest of all commands and when you access the container via **exec**. Adding a non-root user called **btx** without a password and remove some of the basic Linux commands like **ls** , ** cat** , **mv** , **find** and **cd**. Now no need to copy the whole project 🥲, just that tiny binary! and that's it. We know that the app depends on a **.env** file so we copy it inside the container. 


Exposing the container port is dependent on the port number on the **.env** file. Now all artifacts are in the right place, so now let's change the directory ownership to the **btx** user and activate that user to be the default one when we access the container via exec. The final command is to start the app as a single process on the container.

Time to **cook**  😊

Build the docker image : 

```
$ docker build -t node-api-crud:v1 . --no-cache

```
Run the docker image :

```
$ docker run -p 3000:3000 --name node-rest  node-api-crud:v1
Server started on port 3000...
Mongoose connected to db...
Mongodb connected....

```
We can now test the app and start inserting data and getting them via curl via **localhost:3000** 

# Final thoughts 

To ensure more security test try to use : 

* **npm audit fix** to fix any dependency vulnerability.
* **Anshore** for scanning the docker image and check if there is any critical CVE on the system packages.
* **Trusted images** from official docker hub repos, or make your own.
* **Sonarqube** to make some **SAST** and discover potential security issues on your code.

## &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;You build it, you run it, you secure it