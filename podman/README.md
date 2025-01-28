# Podman Commands

## Build an image called app-liberty-server from Dockerfile in current directory
podman build -t app-liberty-server .

## Run a container called a using app-liberty-server image
podman run -it -p 9085:9080 -p 9445:9443 app-liberty-server

## Stop a container by container-name
podman stop app-liberty-server

## Stop a continer by container ID
### Find the container id
podman ps

### Stop the container
podman stop container-id
