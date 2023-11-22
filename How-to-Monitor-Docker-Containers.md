(Version >= 1.18.0)

Docker container status can be queried by Uptime Kuma. How you access the Docker daemon differs depending on how you are running Uptime Kuma, and where the Docker daemon is running.

## Local Docker daemon
If you are running Docker and Uptime Kuma on the same host, the user/container that Uptime Kuma is running in needs permissions to access the Docker socket, which usually lives at `/var/run/docker.sock`.

### Uptime Kuma non-Docker
Every system is different, and how to grant permissions to the Uptime Kuma user to the Docker socket can vary and is largely out of scope of this article. The most straightforward way is to add the Uptime Kuma user to the `docker` group. This should work on most systems:

```bash
addgroup <uptime-kuma-user> docker
```

Refer to your system's man pages for the correct command syntax.

### Uptime Kuma Docker
By default, a docker container is self-contained, which means Uptime Kuma cannot access your host. You need to bind the `/var/run/docker.sock` to your container.

If using `docker run`, add this to your command line:
```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

Or add this to your `docker-compose.yml`:
```yml
volumes:
   - /var/run/docker.sock:/var/run/docker.sock
```

## Remote Docker daemon
Regardless of how you are running Uptime Kuma, you must expose the Docker daemon over a TCP port in order to access it from another host. The primary documentation for achieving this is available [here](https://docs.docker.com/config/daemon/), but the steps specific to Uptime Kuma are documented below.

### Update Docker configuration and Restart Docker
Update the daemon configuration located at `/etc/docker/daemon.json`. Any pre-existing parameters should be kept.

The TCP connection can be insecure or TLS secured using client certificate authentication. You should always be using certificate authentication unless you are in a closed network. **Exposing an unauthenticated Docker daemon socket to the Internet will allow anyone to connect to your server and run commands as root.**

Insecure:
```json
{
   "hosts": ["unix:///var/run/docker.sock", "tcp://<host IP address>:2375"]
}
```

Secure:
```json
{
   # Secure option using client certificate authentication
   "tls": true,
   "tlscert": "/var/docker/server.pem",
   "tlskey": "/var/docker/serverkey.pem",
   "hosts": ["unix:///var/run/docker.sock", "tcp://<host IP address>:2376"]
}
```

Restart the daemon after making changes. For systems running systemd, use `sudo systemctl restart docker.service`. OpenRC and similar systems may use `sudo service docker restart`.

Check the status of the service to ensure it is running. Replace `restart` with `status` in the above commands.

If the daemon fails to start, it may be because there are duplicate hosts specified; it is an error to have one appear more than once. Hosts can be specified by both the command line and `daemon.json` file, so your init system is probably specifying its own hosts with `-H` when executing `docker`.

Systemd users can edit the startup configuration using `sudo systemctl edit docker.service`:
```toml
[Service]
# The blank ExecStart is required to clear the current entry point
ExecStart=
# remove the properties that you have added to the daemon.json file, leave all else the same.
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

My original ExecStart was: `ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock`, note the `-H`; it will cause a duplicate property error.


### Differences with Docker Snap

Docker isntalled using Snap has its configuration file elsewhere. Make the same changes as mentioned above but use this file instead: `/var/snap/docker/current/config/daemon.json`

The restart command for systemd is instead `sudo systemctl restart snap.docker.dockerd.service`. Check status with `status` instead of `restart`.

![Screenshot showing the snap docker service working](https://github.com/louislam/uptime-kuma/assets/642149/8494c876-5580-4f87-9ceb-9a5974f1c977)

### Add certificates to Uptime Kuma

If you are going to be using TLS to secure the connection (very recommended)

## Add Docker to Uptime Kuma
Add a new Docker host and choose what type of Docker connection to make. Specify the socket file path or IP address of the host and the TCP port you exposed, as seen below.

![Docker host monitor](img/docker-host.png)

**Configuring certificates for Docker TLS connection**

Assuming you have already properly configured your remote docker instance to listen securely for TLS connections as detailed [here](https://docs.docker.com/engine/security/protect-access/#use-tls-https-to-protect-the-docker-daemon-socket), you must configure Uptime-Kuma to use the certificates you've generated.  The base path where certificates are looked for can be set with the `DOCKER_TLS_DIR_PATH` environmental variable or defaults to `data/docker-tls/`. 

For running uptime-kuma inside docker, mount the parent directory to `/app/data/docker-tls`. 
```
-v /docker-cert:/app/data/docker-tls
```

If a directory in this path exists with a name matching the FQDN of the docker host (e.g. the FQDN of `https://example.com:2376` is `example.com` so the directory `data/docker-tls/example.com/` would be searched for certificate files), then `ca.pem`, `key.pem` and `cert.pem` files are loaded and included in the agent options. File names can also be overridden via `DOCKER_TLS_FILE_NAME_(CA|KEY|CERT)`.

Remember that for Docker, the file path you enter will need to point at the mounted socket file, not the one on the host.

## Related Discussion

- https://github.com/louislam/uptime-kuma/issues/2061
