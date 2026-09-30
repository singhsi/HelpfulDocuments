# Podman Commands

> For concepts and best practices see [README.md](README.md).

#### build an image from the Dockerfile in the current directory
`podman build -t <image-name> .`

#### run a container with port mappings
`podman run -it -p <host-port>:<container-port> <image-name>`

#### list running containers
`podman ps`

#### list all containers (including stopped)
`podman ps -a`

#### stop a container by name
`podman stop <container-name>`

#### stop a container by ID
```
podman ps                  # find the container ID
podman stop <container-id>
```

#### remove a container
`podman rm <container-name>`

#### list all local images
`podman images`

#### remove an image
`podman rmi <image-name>`

#### view logs for a container
`podman logs <container-name>`

#### open a shell inside a running container
`podman exec -it <container-name> /bin/bash`

---

## Updating the Registries Config

Use this when pulling from a private or insecure registry (e.g. `icr.io`).

```
# 1. find your podman machine name
podman system connection list

# 2. open a shell in the podman machine as root
podman machine ssh --username root <machine-name>

# 3. edit the registries config
vi /etc/containers/registries.conf

# 4. in vi, press Shift+G to go to the end, then o to add a new line and enter insert mode
# 5. add the following block
[[registry]]
location = "icr.io"
insecure = true

# 6. save and exit: Esc then :wq

# 7. exit the podman machine shell
exit
```
