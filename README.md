Grist Omnibus on IONOS Docker server 
====================================

This is the recipe to install Grist on a small IONOS server with Docker pre-installed.

The installation is largely based on [GristLabs grist-omnibus](https://github.com/gristlabs/grist-omnibus/blob/main/README.md).

Grist-omnibus is a repository to install Grist on a server quickly with authentication and certificate handling set up out of the box.

Here we table the specific server and Grist settings we used to get Grist running.

# Cloud Server settings

## Configuration

|       |                |
|-------|----------------|
| Type: | Cloud Server S |
| CPU:  | 1 vCore        |
| RAM:  | 1 GB           |
| SSD:  | 40 GB          |

Although Docker is a light-weight application, we needed a minimum RAM size of 2 GB.

## Data Center Region
The server is based in Berlin, Germany.

## Docker pre-installed
We use a IONOS server with Docker pre-installed. 

## Additional software

For making local changes on the server and to check the status of the server, we use some extra software.
Please run/install:

```
yum install htop
yum install vim
```

## Firewall policies

The configuration of the firewall is set as follows

| Action  | Allowed IP  | Protocol  | Port(s)  | Description  |
|---------|-------------|-----------|----------|--------------|
| Allow   | All         | TCP       | 22       |              |
| Allow   | All         | TCP       | 80       |              |
| Allow   | All         | TCP       | 443      |              |
| Allow   | All         | TCP       | 993      |              |
| Allow   | All         | TCP       | 995      |              |
| Allow   | All         | TCP       | 8484     | Grist        |

Ideally, port 80 would not be needed as we host Grist over HTTPS (port 443).

Port 8484 would also not be needed since we want traffic to Grist to go over HTTPS, and not directly.
This might be handy though for testing purposes.

## Services

The IONOS server has a few applications pre-installed, we want to disable some since we don't use them.
Services are maintained on the server with systemd.

To see a list of all running services run: `systemctl --type=service --state=running`.

With `ps axjf` you can get a nice overview of all running processes on the whole server. Running `htop` gives you a nice overview of the current status (cpu and memory usage) of the server.

We want to disable MariaDB and PHP-FPM:

```
systemctl stop mariadb
systemctl disable mariadb
systemctl stop php-fpm
systemctl disable php-fpm
```

Sometimes stopping a service doesn't work. You can then try to kill the process via `kill -9 PID`. The PID can be found with `ps axjf`.

## Grist service

We want to have Grist running as a systemd service too. This repository comes with a `grist-ombibus.service` file just for that.
Please adjust the file to your liking and put the file on the server in `/etc/systemd/system`.

Next run:

```
systemctl daemon-reload
systemctl enable grist-omnibus
systemctl start grist-omnibus
```

This should start the service.
You can check (tail) the logs/output with:

```
journalctl -u grist-omnibus.service -f
```
## Grist upgrade

To upgrade Grist, you need to do the following steps:

1. Make a data backup
2. Backup your current Docker images
3. Pull new Docker images
4. Restart Grist

### 1. Make a data backup

TODO

### 2. Backup your current Docker images

Run:

```
docker image ls
```

You will get the following overview:

```
REPOSITORY                        TAG               IMAGE ID       CREATED         SIZE
gristlabs/grist                   latest            8b1ad55973c0   4 hours ago     1.01GB
gristlabs/grist-omnibus           latest            79751cdb802c   13 hours ago    1.22GB
gristlabs/grist-omnibus           backup-20240501   6823d27e4530   3 months ago    1.16GB
gristlabs/grist                   backup-20240501   76b3f1c5d1df   3 months ago    950MB
docker                            latest            cf951b77e884   3 months ago    331MB
python                            latest            fc7a60e86bae   4 months ago    1.02GB
hello-world                       latest            d2c94e258dcb   12 months ago   13.3kB
traefik                           v2.6              22c6901de2be   23 months ago   102MB
thomseddon/traefik-forward-auth   2                 35e1f839c20f   3 years ago     14.2MB
```

Notice `gristlabs/grist` and `gristlabs/grist-omnibus` having a backup from a previous upgrade. The current version runs on "latest".

To backup the current latest tags run:

```
docker tag gristlabs/grist-omnibus:latest gristlabs/grist-omnibus:backup-$(date +%Y%m%d)
docker tag gristlabs/grist:latest gristlabs/grist:backup-$(date +%Y%m%d)
```

Check `docker image ls` again to see the new tags.

### 3. Pull new Docker images

Run the following:

```
docker pull gristlabs/grist-omnibus:latest
docker pull gristlabs/grist:latest
```

### 4. Restart Grist

First find the current container ID and next stop it:

```
docker ps
```

```
CONTAINER ID   IMAGE                     COMMAND                  CREATED          STATUS          PORTS                                                                                NAMES
b8d087e7dab4   gristlabs/grist-omnibus   "docker-entrypoint.s…"   31 minutes ago   Up 31 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 8484/tcp   amazing_ride
```

```
docker stop b8d087e7dab4
```

Or use the one-liner:

```
docker stop $(docker ps -q --filter "ancestor=gristlabs/grist-omnibus" | head -n 1)
```

Next, start Grist again:

```
cd grist-omnibus/
./start.sh
```

# Grist settings

Here's the minimal configuration you need to provide.
 * `EMAIL`: an email address, used for Let's Encrypt and for
   initial login.
 * `PASSWORD`: optional - if you set this, you'll be able to
   log in without configuring any other authentication
   settings. You can add more accounts as `EMAIL2`,
   `PASSWORD2`, `EMAIL3`, `PASSWORD3` etc.
 * `TEAM` - a short lowercase identifier, such as a company or project name
   (`grist-labs`, `cool-beans`). Just `a-z`, `0-9` and
   `-` characters please.
 * `URL` - this is important, you need to provide the base
   URL at which Grist will be accessed. It could be something
   like `https://grist.example.com`, or `http://localhost:9999`.
   No path element please.
 * `HTTPS` - mandatory if `URL` is `https` protocol. Can be
   `auto` (Let's Encrypt) if Grist is publically accessible and
   you're cool with automatically getting a certificate from
   Let's Encrypt. Otherwise use `external` if you are dealing
   with ssl termination yourself after all, or `manual` if you want
   to provide a certificate you've prepared yourself (there's an
   example below).

The minimal storage needed is an empty directory mounted
at `/persist`.

So here is a complete docker invocation that would work on a public
instance with ports 80 and 443 available:
```
mkdir -p /tmp/grist-test
docker run \
  -p 80:80 -p 443:443 \
  -e URL=https://cool-beans.example.com \
  -e HTTPS=auto \
  -e TEAM=cool-beans \
  -e EMAIL=owner@example.com \
  -e PASSWORD=topsecret \
  -v /tmp/grist-test:/persist \
  --name grist --rm \
  -it gristlabs/grist-omnibus  # or grist-ee-omnibus for enterprise
```

And here is an invocation on localhost port 9999 - the only
differences are the `-p` port configuration and the `-e URL=` environment
variable.
```
mkdir -p /tmp/grist-test
docker run \
  -p 9999:80 \
  -e URL=http://localhost:9999 \
  -e TEAM=cool-beans \
  -e EMAIL=owner@example.com \
  -e PASSWORD=topsecret \
  -v /tmp/grist-test:/persist \
  --name grist --rm \
  -it gristlabs/grist-omnibus  # or grist-ee-omnibus for enterprise
```

If providing your own certificate (`HTTPS=manual`), provide a
private key and certificate file as `/custom/grist.key` and
`custom/grist.crt` respectively:

```
docker run \
  ...
  -e HTTPS=manual \
  -v $(PWD)/key.pem:/custom/grist.key \
  -v $(PWD)/cert.pem:/custom/grist.crt \
  ...
```

Remember if you are on a public server you don't need to do this, you can
set `HTTPS=auto` and have Traefik + Let's Encrypt do the work for you.

If you run the omnibus behind a separate reverse proxy that terminates SSL, then you should
`HTTPS=external`, and set an additional environment variable `TRUSTED_PROXY_IPS` to the IP
address or IP range of the proxy. This may be a comma-separated list, e.g.
`127.0.0.1/32,192.168.1.7`. See Traefik's [forwarded
headers](https://doc.traefik.io/traefik/routing/entrypoints/#forwarded-headers).

You can change `dex.yaml` (for example, to fill in keys for Google
and Microsoft sign-ins, or to remove them) and then either rebuild
the image or (easier) make the custom settings available to the omnibus
as `/custom/dex.yaml`:

```
docker run \
  ...
  -v $PWD/dex.yaml:/custom/dex.yaml \
  ...
```

You can tell it is being used because `Using /custom/dex.yaml` will
be printed instead of `No /custom/dex.yaml`.
