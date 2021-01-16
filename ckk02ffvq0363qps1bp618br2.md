## Scale your containers with ansible on a non kubernetes/swarm environment

> Scaling containers on Kubernetes with Replicatsets or scale services with docker swarm is a very easy task! but what about scaling containers in a non managed environment, when you have high traffic, absolutely you need to scale and load balance the traffic among multiple containers. In this article, we will use Ansible and nginx to scale containers.


# Setting up the environment

Before starting make sure you have the required packages: 

- Ansible 2.9: `pip install ansible`
- Docker latest : follow this [link](https://phoenixnap.com/kb/how-to-install-docker-on-ubuntu-18-04) to install docker 
- Docker python package installed on the remote host: `pip install docker`
- A ubuntu machine (18.04)

# Let's Code!

In this section, we will attach the puzzle pieces, we will test everything locally, but the actual aim of this setup is to integrate it into the pipeline, so we can put the scale number and deploy it if you don't want to make it in the CI/CD pipeline, it's ok you can just launch the playbook against the server and scale your container (don't forget the container name), automatically will scale the containers and update the nginx config.


## Inventory file 
The inventory file will hold our variables and the target where we can  execute the task, it contains two host children (network and clean)  derived from the main host (local) 

```toml
[local]
localhost ansible_connection=local
[local:vars]
scale=5
c_name=<container-name>
h_name=<container-name>
subnet=172.16.0                                           
network_name=network
image_name=<image-name>
[network:children]
local
[clean:children]
local
```
The most important variable is `scale`, it's the number of containers that will start in the host, the other variables are about the container name, the image, and the subnet where the containers will run.

## Ansible playbook 

The ansible-playbook will hold all the logic of what we will do here. the inventory files variables will be accessible to the playbook 
### Creating the network

This task will create the network with the specified subnet and the gateway, note that the network will be created if doesn't exist, otherwise, the task will be skipped 
```yaml
- hosts: network
  tasks:                                                     
  - name: Create docker network
    docker_network:
      name: "{{ network_name }}"
      ipam_config:
        - subnet: "{{ subnet }}.0/16"
          gateway: "{{ subnet }}.1"
```

### Getting the last scale number

Based on the image name we can get the number of containers actually running, to do that we used a shell script the gather that information. The `set_fact` will create a variable dynamically that will be accessible to the whole playbook by registering the output of the script and get the stdout as a JSON format. 

```yaml

  - name: get last scale
    script: ./get_cont_num.sh {{ image_name }}
    register: last_scale

  - set_fact: 
      last_scale={{ last_scale.stdout }}
  
  - debug: var=last_scale
```
We can debug the variable content by adding the `debug` keyword and print it out.
```bash
#!/bin/bash 

docker ps -f ancestor=$1 --format '{{.Names}}' | wc -l
```

This bash script will give us names like this `name_1,name2 ..etc` so we can count it with the word count command `wc` the line option.
### Starting the container 

Starting the container with the actual number specified in the `scale` variable, looping into it with the `with_sequence` and index the counter with `item`, For the `ipv4_address` it starts from `2` because the first one is for the gateway. 
```yaml 
 - name : starting containers
    docker_container:
      name: "{{ c_name }}_{{ item }}"
      image : "{{ image_name }}"
      pull: yes
      restart_policy: always
      hostname: "{{ h_name }}_{{ item }}"
      networks:
        - name: "{{ network_name }}"
          ipv4_address: "{{ subnet }}.{{ 1+item|int }}"
      purge_networks: yes
    with_sequence: start=1 end="{{ scale }}"
```
If you have many containers on the network make sure to separate the IPs (host ID) with al least 5 like this : 
```
Frontend : 172.16.0.2
Backend: 172.16.0.8
...
```
To be able to scale each container and don't make an IP conflict when scaling.


### Removing container
In this task, we will remove the unneeded containers, it starts only when the `scale number > last scale` otherwise the task will be skipped. but we have to make things clear on the `with_sequence` logic because it will be compiled but not executed when skipping.

```yaml
  - name : down-scaling uneeded containers 
    docker_container: 
      name: mongo_{{ item }}
      state: absent
    with_sequence: start="{{ last_scale|int if (last_scale|int - scale|int)|abs == 1|int  or last_scale|int == scale|int  or scale|int > last_scale|int else 1+scale|int  }}" end="{{ last_scale }}" 
    # last_scale=4 scale=2
    # last_scale=3 scale=3
    # last_scale=3 scale=4 
    # last_scale=2 scale=4
    when: scale|int < last_scale|int
```
The logic says that if : 
==> return last_scale `if` 
* Positive(last_scale - scale) == 1 `or`
* last_scale == scale `or`
* scale > last_scale `else`
* return 1 + scale   
This logic can ensure removing the right number of containers without mistakes 

### Getting containers IPs
In this task we will get all IPs of our scaled containers. The set fast will contain all IPs by filtering the output from `register` by splitting with `r\n` to get a loopable list 
```yaml
  - name: generate  ip 
    script: ./get_name_ip.sh {{ scale }}
    register: last_scale
    
  - set_fact: 
        list_ip={{ last_scale.stdout.split("\r\n")|list }}
```
Depending on the scale number we can loop over the container names (name_1,name_2) and get all IPs associated with every container.

```bash
#!/bin/bash
scale=$1

for i in $(seq $scale) 
do
docker inspect container_$i | jq ".[].NetworkSettings.Networks.network.IPAddress" -r
done
```

### Generating nginx.conf file
Thanks to the `set_fact` all variables that are created in the playbook dynamically will be exposed by default to the `jinja2` template so we can use it easily. 
```yaml
  - name : generate file 
    template : 
        src: example.domaine.com.conf.j2
        dest: example.domaine.com.conf
```
The `template` module will generate a new `nginx.conf` depends on the scale number, I mean if we have `scale=1` we will generate a simple configuration, otherwise, we will make the load balancing configuration with the containers IPs 
```nginx
{% if  scale > 1 %}
upstream manager {
{% for i in range(scale) -%}
server http://{{ list_ip[ i ] |ipaddr }};
{% endfor %}
}
server {
        listen 80;
        listen [::]:80;
        server_name admin.domaine.com;
        return 301 https://$host$request_uri;
}
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name admin.domaine.com;
        ssl on;
        ssl_certificate /etc/letsencrypt/live/admin.domaine.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/admin.domaine.com/privkey.pem;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        location / {
                include /etc/nginx/extra.d/caching_ht.conf;
                proxy_pass http://manager;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade; # allow websockets
                proxy_set_header X-Forwarded-For $remote_addr; # preserve clien$
                proxy_set_header Host $remote_addr; # preserve client IP
        }
}
{% else %}
server {
        listen 80;
        listen [::]:80;
        server_name admin.domaine.com;
        return 301 https://$host$request_uri;
}
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name admin.domaine.com;
        ssl on;
        ssl_certificate /etc/letsencrypt/live/admin.domaine.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/admin.domaine.com/privkey.pem;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        location / {
                include /etc/nginx/extra.d/caching_ht.conf;
                proxy_pass http://{{ list_ip[0]|ipaddr }};
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade; # allow websockets
                proxy_set_header X-Forwarded-For $remote_addr; # preserve clien$
                proxy_set_header Host $remote_addr; # preserve client IP
        }
}
{% endif %}
```
For the SSL configuration, you can make sit easily by running the certbot command and get your certificate, note that this configuration is applied to the frontend containers because we have SSL, you can apply it to your backend easily and update the nginx configurations easily.
### Applying the nginx configuration

This task is trivial, we just checking nginx syntax and reloading configurations, if you want you can change the reload by restarting nginx.

```yaml
   - name: Verify Nginx config
     become: yes
     command: nginx -t
     changed_when: false

   - name: reload Nginx config
     become: yes
     command: nginx -s reload
     changed_when: false
```

### Cleanning

Clearing unused containers and volumes or images with no tags is a best practice for making your server in a good mood :) 

```yaml
- hosts : clean
  gather_facts : no 

  tasks: 

  - name: Removing exited containers
    shell: docker ps -a -q -f status=exited | xargs --no-run-if-empty docker rm --volumes
  - name: Removing untagged images
    shell: docker images | awk '/^<none>/ { print $3 }' | xargs --no-run-if-empty docker rmi -f
  - name: Removing volume directories
    shell: docker volume ls -q --filter="dangling=true" | xargs --no-run-if-empty docker volume rm

```


I'm integrating this solution on my CI/CD because to scale when I push my code, it's simple and needs more work to be generic and reliable, you can find all the files on [Github repository](https://github.com/hatembentayeb/AnsibleDockerScaler). The demo gif on the repository I test it with mongo, I know that we cannot scale databases like this actually, I'm jus treating it as a container that's it.