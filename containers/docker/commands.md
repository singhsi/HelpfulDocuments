# Docker Commands

> For concepts and best practices see [README.md](README.md).

---

## System & General

#### get help with Docker or subcommands
`docker --help`

#### display system-wide information
`docker info`

#### start the Docker daemon
`docker -d`

---

## Images

#### build an image from a Dockerfile
`docker build -t <image_name> .`

#### build an image from a Dockerfile without using cache
`docker build -t <image_name> . --no-cache`

#### list local images
`docker images`

#### delete an image
`docker rmi <image_name>`

#### remove all unused images
`docker image prune`

---

## Docker Hub & Registries

#### log in to Docker Hub or private registry
`docker login -u <username>`

#### search Docker Hub for an image
`docker search <image_name>`

#### pull an image from Docker Hub
`docker pull <image_name>`

#### publish an image to Docker Hub
`docker push <username>/<image_name>`

---

## Containers

#### create and run a container from an image with a custom name
`docker run --name <container_name> <image_name>`

#### run a container and publish a container's port(s) to the host
`docker run -p <host_port>:<container_port> <image_name>`

#### run a container in the background (detached mode)
`docker run -d <image_name>`

#### start an existing stopped container
`docker start <container_name>`

#### stop a running container
`docker stop <container_name>`

#### list currently running containers
`docker ps`

#### list all containers (running and stopped)
`docker ps --all`

#### inspect a container for detailed configurations and metadata
`docker inspect <container_name>`

#### view live container resource usage stats
`docker container stats`

#### fetch and follow the logs of a container
`docker logs -f <container_name>`

#### open an interactive shell inside a running container
`docker exec -it <container_name> sh`

#### remove a stopped container
`docker rm <container_name>`
