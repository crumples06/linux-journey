# Week-9
*10/08/26 - 16/08/26*

## Docker
I now feel like I've learned adequate amount in linux to move forward. There are many choices from here like docker, advanced scripting, servers, etc. I chose to learn docker first.

Docker is used to containerize applications, making it so that they run in a consistent  environment on any machine. 
Docker uses a client server architecture. The Docker client talks to the Docker Daemon, which builds, runs, and manages containers. They communicate through a REST API via UNIX sockets or a network interface.

Components of docker:
- Docker Engine: Handles creation and management of containers
- DockerFile: File that describes the steps to create an image quickly
- Docker Image: Template that is used for creating containers, containing application code and dependencies
- Docker Hub: Cloud based repository used for finding and sharing docker images
- Docker Registry: Storage distribution system for docker images

## Docker CLI
- `docker run` is used to create and start a container from an image
- `docker ps` lists all the containers
- `docker pull` is used to download images from registery
- `docker images` shows all the images in the pc. 
I first ran `docker images` which showed ubuntu as the only image. Then i ran `docker pull mysql`, after that the list showed ubuntu and mysql. I can remove an image by running `docker rmi image-name`.

There are a lot of commands and flags for docker in CLI, they are all listed and explained nicely in `docker --help`.

`docker logs` shows logs for a particular container. I ran `docker logs unruffled_snyder` and it showed me the logs for the ubuntu container that was running. If i use the flag `-f` it shows the logs in real time.

`docker exec` is used to execute commands in an running container. I had a ubuntu container running in a terminal and i used another terminal to run the `docker exec` command to run a simple echo command in the ubuntu container.
I was expecting to have that echo command logged, so i ran the logs command but it was not logged. 
I found out that the log command only shows/logs the commands from PID 1, which is the first terminal i had opened on which the container was running. This is supposed to be a feature as the exec command is to be used for quick debugging and hence it should not clutter the important logs from PID 1.

`docker stop` and `docker kill` both stop a container, the difference is that 'stop' first sends a signal requesting for the container to stop and then waits for a grace period and if the container has not been stopped then it force closes it, while 'kill' directly force closes a container.


## Dockerfile
It is a plain text file containing instructions to create a docker image. Each instruction creates a new layer in the docker image.
Key Components:
- Base Image: Starting point for a docker image
- Application code and dependencies: Code is added and dependencies are installed
- commands and configurations: Instructions to execute commands, set environment variables, and expose ports

Common Instructions:
- `FROM`: Specifies a base. example: `FROM python:3.9`
- `COPY`: Used to copy files from host system to the container
- `RUN`: Executes commands during the build process. 
- `CMD`: Specifies the default command to run when the container starts. ex: `CMD ["python3", "app.py"]`
- `WORKDIR`: sets the working directory inside the container
- `EXPOSE`: Documents the port the container listens on. ex: `EXPOSE 8000`


Dockerfile for my script `processSnapshot`:

FROM alpine:3.20
RUN apk add --no-cache bash procps util-linux
WORKDIR /app
COPY processSnapshot  /app/processSnapshot
RUN chmod +x /app/processSnapshot
CMD ["bash", "/app/processSnapshot"]

If i run the script in the container made using the image from the above dockerfile, the script will not actually monitor my laptops processes. It will work on the container's PID namespace.
For the script to show my host machine's processes, i need to make the container use the host's PID namespace so that it can see the host's processes.
For that i need to run the container using `docker run --rm -it --pid=host process-snapshot`.
This weakens the container isolation. This i think is better used in a container which is doing other things, so it will monitor those processes in the container.

## Docker image layers
Each instruction in Dockerfile that modifies the filesystem creates a new layer. The layers are built on top of each other. Layers use functionality from below layers. 
If a layer changes, all layers after it are rebuilt. That is why the least frequently changed instructions are put first (like installing dependencies) and most frequently changed ones are last (like copying application code).

I can also built on top of an existing image to make another image, like when i do `FROM python:3.12-slim` i am using a base image containing python with debian slim. These base images can be anything, for example i can make an application as a base image and then use that image in other images to add functionality to the application without having to make the application again and again in each image.

## Docker Volume
Volumes are persistent data stores for containers, created and managed by Docker, it's stored within a directory on the Docker host. Volumes are not a good choice if you need to access the files from the host, as the volume is completely managed by Docker.
Volumes are often a better choice than writing data directly to a container, because a volume doesn't increase the size of the containers using it. Using a volume is also faster; writing into a container's writable layer requires a storage driver to manage the filesystem.

## Docker Bind Mounts
When you use a bind mount, a file or directory on the host machine is mounted from the host into a container. 
Bind mounts are appropriate for the following types of use case:
- Sharing source code or build artifacts between a development environment on the Docker host and a container.
- When you want to create or generate files in a container and persist the files onto the host's filesystem.
- Sharing configuration files from the host machine to containers

## python server in docker
I wanted to have a server running in a container and see what can i do with it. I made a simple python server and a basic docker file:
```
FROM python
COPY server.py /server/
WORKDIR /server
CMD python3 server.py
```

I built the image using `docker build .`, i ran this command in the directory where these files were stored.
When i went to see my images using `docker images`, the output wasn't showing my recently created image, but i could see it in docker desktop. So i tried `docker images -a`, now i could see the images, here i notices that the image name was <none> as i forgot to name while building. So i named the image by `docker image-id python-server:latest`. Now when i ran `docker images` i could see the image.

I ran the image i.e. making a container by `docker run -it python-server`. The `-it` flag runs it interactively with a terminal.
If i want to interact with the server i need to attach a port when running the container by, `docker run -it -p 8080:8080 python-server`, then i can interact with it by `curl http://localhost:8080`.

## python server in docker
I wanted to have a server running in a container and see what can i do with it. I made a simple python server and a basic docker file:
```
FROM python
COPY server.py /server/
WORKDIR /server
CMD python3 server.py
```

I built the image using `docker build .`, i ran this command in the directory where these files were stored.
When i went to see my images using `docker images`, the output wasn't showing my recently created image, but i could see it in docker desktop. So i tried `docker images -a`, now i could see the images, here i notices that the image name was <none> as i forgot to name while building. So i named the image by `docker image-id python-server:latest`. Now when i ran `docker images` i could see the image.

I ran the image i.e. making a container by `docker run -it python-server`. The `-it` flag runs it interactively with a terminal.
If i want to interact with the server i need to attach a port when running the container by, `docker run -it -p 8080:8080 python-server`, then i can interact with it by `curl http://localhost:8080`.


