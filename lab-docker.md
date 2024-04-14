# Lab - Docker

## 1. Introduction

Move inside the directory with `Dockerfile`.

### Build an image

```shell
docker build -t app:v1 .
docker build -t registry.deti/nmec/app:v1 . # For registry
docker build -t app:v1 -f Dockerfile.app . # Using a specification
```

### Push an image to a registry

```shell
docker push registry.deti/nmec/app:v1
```

Note: if `deti.registry` cannot be found, add the entry `193.136.82.36 registry.deti` to `/etc/hosts`.

### Run a container

```shell
docker run -ti --name app-v1 -p 80:8080 app:v1 
docker run -ti --name app-v1 registry.deti/nmec/app:v1 # From registry
```

Note: `-p` maps port outside the container `80` to internal port `8080`.

### Go inside a container

```shell
docker exec -ti b6 /bin/sh
```

Note: in this case, `b6` is the beggining of the container ID.

## 2. Persistence

Storing static data inside a container is a bad practice - containers are stateless.

We should define a volume outside of the container, that the container can access. It is usually used to store static and configuration files. Updates to the volumes are reflected in the container imediately.

**Bind Mount:** map a local path to a path inside a container.

- used only for testing.

- defined by writing a <u>path</u>.

**Volume Mount:** managed by Docker and mapped to a path inside the container.

- unspecified block of data that can be attached to a container.

- defined by writing a <u>name</u>.

### Volume commands

```shell
docker volume ls
docker volume create --name www
docker volume inspect www
```

### Run a container with a bind mount

```shell
docker run -v $(pwd)/www:/var/www app:v2
docker run -v $(pwd)/www:/var/www:ro app:v2 # Read Only
```

This command runs a container where the directory `$(pwd)/www` (or `./www`) in the host is mapped inside the container in the directory `/var/www`.

### Run a container with a volume bind

```shell
docker run -v www:/var/www app:v2
docker run -v www:/var/www:ro app:v2 # Read Only
```

This command runs a container where a Docker managed volume `www` in the host is created and mapped inside the container in the directory `/var/www`.

## 3. Docker Compose

Used to create and orchestrate services based on images. Containers in different docker compose definitions can communicate by defining a network.

Warning 1: the <u>name of the project</u> for docker compose is by default the <u>name of the directory</u> where docker compose is run. If there is another deployment running under the same name, `docker compose down` might indicate orphans exist.

Warning 2: <u>never use IP addresses</u> to reference other services, always use their <u>name</u>. There is always a DNS inside the network.

### Docker compose commands

```shell
docker-compose up # Old
docker compose up # New
docker compose up --build # Only builds services with "build" tag
```

## 4. Docker Swarm

Container orchestration tool for clustering and scheduling Docker containers. We skipped it since we will use kubernetes for this.

## 5. Configuration

Configuration files can be associated with specifc services inside `docker-compose`. They can be used to replace default configurations for a same image, and changes to them are reflected imediately inside the container, since the files are mapped to directories inside them.

```yml
version: "3.8""
services:
  ...
  nginx:
    image: nginx
    configs:
      - source: nginx_conf
        target: /etc/nginx/ngxin.conf # Overrides configs in container
configs:
  nginx_conf:
    file: nginx.conf # Config file outside container
```

## 6. Secrets

Secrets are sensitive configurations, such as SSL certificates for HTTPS, access credentials or API keys. Like config files, these can be associated with specific services inside `docker-compose`. Unlike config files, secrets are <u>encripted at rest and in transit</u>, making them suitable for storing sensitive information.

```yml
version: "3.8"
services:
  ...
  nginx:
    image: nginx
    ...
    secrets:
      - source: cert
        target: /etc/ssl/private/cert.pem
      - source: key
        target: /etc/ssl/private/key.pem
secrets:
  cert:
    file: cert.pem
  key:
    file: key.pem
```
