# DNCS-DOCKER-SWARM

## Table of contents:
- [DOCKER-MACHINE CREATION](#docker-machine-creation)
- [SWARM INITIALIZATION](#swarm-initialization)
- [STACK DEPLOYMENT](#stack-deployment)
  - [Create the application](#create-the-application)
  - [Create the registry](#create-the-registry)
  - [Create the stack](#create-the-stack)
- [SCALE THE SERVICE](#scale-the-service)
- [SWARMPIT](#swarmpit)
- [TASK DISTRIBUTION](#task-distribution)
- [LOAD BALANCING](#load-balancing)
- [MIGRATION](#migration)

## DOCKER-MACHINE CREATION

To activate a docker-swarm we need at least one Linux host with Docker Engine installed. We have used Docker Machine to create the virtual hosts with Docker Engine. The commands used are:

`docker-machine create --driver virtualbox *host-name*`

`docker-machine env *host-name*`

`eval $(docker-machine env *host-name*)`

The virtual hosts we created are:
- manager1
- worker1
- worker2

We can see the IP of the hosts using: `docker-machine ip *host-name*`

## SWARM INITIALIZATION
To initialize the swarm we have to start the manager1 and log in, using the commands:

`docker-machine start manager1`

`docker-machine ssh manager1`

Once we are in the manager1, we have to init the swarm:

`docker-machine ssh manager1`

`docker swarm init --advertise-addr *manager-ip*`

The output of the `docker swarm init` is a command, that should be like:
```
Swarm initialized: current node (b3p06r81asv6o0p0allos5uwm) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1itkiood63gxersy91bire2s02qet7y7gwtm5vzw59dxr1vyak-9treo6spxv8o9fyjwbpbnliqt 192.168.99.103:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
To add a worker to the swarm we should ssh into the two workers and run the `docker swarm join` command followed by the token.

Into the manager1, you can see the node of the swarm with the command: `docker node ls`


## STACK DEPLOYMENT
We have decided to deploy an application stack to the swarm. Then we created a registry to distribute the images to all the nodes, pushed the images on it and created the stack.

### Create the application
To create the application we have created the files required into a directory in the manager1:
- app.py
- requirements.txt
- Dockerfile
- docker-compose.yml

To create the image' app ran the command: `docker-compose up -d`

Then we can see that the app is running using: `docker-compose ps` to see the processes running
Then we can test the app using: `curl http://localhost:8000`

The output of the curl should be: `Hello World! I have been seen 1 times.`

We can test the web server also searching the `http://*manager1-ip*:8000` url on your browser.

### Create the registry
To distribute the images to all the nodes, aregistry is necessary: `docker service create --name registry --publish published=5000,target=5000 registry:2`

We can check the status of the service using: `docker-service ls`

And check if it's working using: `curl http://localhost:5000/v2/`
The output should be: `{}`

Then we distributed the web app’s image across the swarm using: `docker-compose push`


### Create the stack
We created the stack using: `docker stack deploy --compose-file docker-compose.yml stackdemo`

We can check if the stack is running using: `docker stack services stackdemo`
and test the app with curl: `curl http://localhost:8000`

The output should be: `Hello World! I have been seen 2 times.`

## SCALE THE SERVICE
We can change the scale of a service using: `docker service scale *service-name*=5`
and check the distribution of the tasks using: `docker service ps *service-name*`

The output should be:
```

ID                  NAME                  IMAGE                             NODE                DESIRED STATE       CURRENT STATE           ERROR                         PORTS
urp9jgjpzq2l        stackdemo_web.1       127.0.0.1:5000/stackdemo:latest   worker1             Running             Running 3 hours ago                                                                   
1j39dx7xyp48        stackdemo_web.2       127.0.0.1:5000/stackdemo:latest   manager1            Running             Running 3 hours ago                                    
bag3dj44t5q2        stackdemo_web.3       127.0.0.1:5000/stackdemo:latest   worker2             Running             Running 3 hours ago                                                                
mnm8e0w6aljj        stackdemo_web.4       127.0.0.1:5000/stackdemo:latest   worker1             Running             Running 3 hours ago                                                                    
kcfx7072d85o        stackdemo_web.5       127.0.0.1:5000/stackdemo:latest   worker2             Running             Running 3 hours ago  

```                                                        

## SWARMPIT
We used Swarmpit to monitor the swarm. To install Swarmpit we run on the manager:
```
docker run -it --rm \
  --name swarmpit-installer \
  --volume /var/run/docker.sock:/var/run/docker.sock \
swarmpit/install:1.8
```
Then we can access on Swarmpit with the credentials using `http://*manager-ip*:888` url on the browser.

## TASK DISTRIBUTION
Tasks are distributed on active nodes in the swarm. A node can be set in drain availability to prevent it receiving tasks: `docker node update --availability drain *host-name*`

We can check the availability of the nodes using: `docker node ls`

We can run: `docker service ps *service-name*` to see how the manager updated the tasks.

To return the drained node to an active state run: `docker node update --availability active *host-name*` and change the scale of the service to 0, and then to 5 to update the distribution of the task.

## LOAD BALANCING
The swarm manager uses ingress load balancing to expose the services and make them available externally to the swarm. However we have implemented an external load balancer using Traefik image.

First we created the overlay network using: `docker network create --driver=overlay treafik-net`

Then we created the traefik service:

`docker service create --name traefik --constraint=node.role==manager --publish 80:80 --publish 8080:8080 --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock --network traefik-net traefik:v1.6 --docker --docker.swarmmode --docker.domain=traefik --docker.watch --web`

and updated the stackdemo_web service:

`docker service update --label-add traefik.port=80 --network-add  traefik-net --label-add traefik.frontend.rule=Host:localhost:8000 stackdemo_web`

We can see the web interface of Traefik on the url: `http://*manager-ip:8080*`

## MIGRATION
To create a new image from a container changes we used docker commit command: `docker commit *container-id* *docker-hub-repository*`

Then login: `docker login --username=*username*` and enter the password.

Finally we can do the push: `docker push *docker-hub-repository*`

Now the image is available on Docker Hub and can be pulled and run by everyone and everywhere.

## AUTHORS
This project has been developed by **Lucrezia Ruggeri**, **Sara Layachi**, **Carlotta Neri** and **Elisabetta Rossetti**, as assignment of the 'Design of Network and Communication Systems' course of University of Trento.
