# Liberty Commands

> For concepts and best practices see [README.md](README.md).

---

## Server

#### create a new Liberty server
`<wlp-install>/bin/server create <server-name>`

#### start a server
`<wlp-install>/bin/server start <server-name>`

#### start a server in the foreground (output printed to console — useful for debugging)
`<wlp-install>/bin/server run <server-name>`

#### stop a server
`<wlp-install>/bin/server stop <server-name>`

#### check server status
`<wlp-install>/bin/server status <server-name>`

#### package a server and its apps into a single zip for deployment
`<wlp-install>/bin/server package <server-name> --include=all`

---

## Logs

Liberty logs live at:
```
<wlp-install>/usr/servers/<server-name>/logs/
  messages.log     # main server log — start here when something goes wrong
  console.log      # stdout/stderr from the JVM
  ffdc/            # first failure data capture — detailed crash dumps
```

#### watch the main log in real time
`tail -f <wlp-install>/usr/servers/<server-name>/logs/messages.log`

---

## Features

Features are listed in `server.xml`. You can enable/disable them without restarting from the admin centre, or by editing the file directly.

#### list all installed features
`<wlp-install>/bin/featureUtility find`

#### install a feature
`<wlp-install>/bin/featureUtility installFeature <feature-name>`

---

## Containerised Liberty (Podman / Docker)

#### build a Liberty container image
`podman build -t my-liberty-app .`

#### run a Liberty container
`podman run -d -p 9080:9080 -p 9443:9443 --name my-liberty-app my-liberty-app`

#### open a shell inside a running Liberty container
`podman exec -it my-liberty-app /bin/bash`

#### view Liberty logs from a container
`podman logs my-liberty-app`

#### stop and remove the container
```
podman stop my-liberty-app
podman rm my-liberty-app
```

---

## Useful Links

| Resource | URL |
|---|---|
| WebSphere Liberty Documentation | https://www.ibm.com/docs/en/was-liberty/base?topic=administering-liberty |
| WebSphere Liberty Container Images | https://www.ibm.com/docs/en/was-liberty/base?topic=images-liberty-container |
| Open Liberty Documentation | https://openliberty.io/docs/latest/overview.html |
| WebSphere Migration Toolkit | https://github.com/IBMTechSales/klp-workshop-labs/tree/master/1171-Liberty-Migration-Tools |
| Migration: WAS Traditional → Liberty | https://medium.com/globant/migration-from-ibm-websphere-traditional-server-to-websphere-liberty-server-9ffd7bf997d4 |
| Containerize WebSphere to Liberty (IBM w3) | https://w3.ibm.com/services/lighthouse/videos/123335 |
