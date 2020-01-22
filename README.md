# Traefik v2 with a wildcard SSL certificate for micro services

### This repository contains docker-compose files to deploy Traefik v2 to a Docker Swarm cluster and handle SSL termination and routing.

## Prerequisites
* a Docker Swarm cluster with one manager and a worker node
* a domain
	* Required for Let's Encrypt wildcard SSL certificates
	* A DNS provider (this example uses DigitalOcean but many are supported)
* Docker CE 17.09.0-ce or Docker EE 17.06.2-ee-3 +
* docker-compose latest (v1.25.2) 

## Overview
The aim of this repository is to have a workflow to develop micro services i.e. `my-api` that once deployed to our Swarm cluster they will automatically be available at a subdomain i.e. `https://my-api.mydomain.com` Traefik makes this very easy. Note that if you are only interested in using directory-based routing, (i.e. `https://mydomain.com/my-api`) the current approach is probably an unnecessary overhead.

In Traefik the most common use case for Let's Encrypt is to issue a top level certificate and one for each sub-domain. This approach could be problematic when we have a lot of subdomains or we want to test dynamic micro services due Let's Encrypt API rates. Fortunately, we can use wildcard certificates that cover all subdomains in a given domain and so we can deploy APIs or back-ends that will use the same global wildcard certificate.

To get a wildcard SSL certificate from Let's Encrypt we need to use the DNS challenge method which requires that our domain is handled by one of the supported [DNS providers](https://docs.traefik.io/https/acme) (you can also buy the domain in any Registar and point your NS servers to a supported DNS provider)



## Instructions

The first step is to create 2 networks in our Swarm cluster, `internal` for our micro services and `proxy` for Traefik and other potential external facing services. 

```bash
docker network create --driver=overlay internal --internal
docker network create --driver=overlay proxy
```
Then we need to fill our environment file with the appropriate values. We are using basic auth with a md5 hashed password for the Traefik dashboard at `https://traefik.mydomain.com` 
to generate a valid hash we use `htpasswd` or an online service. Make sure that `$` are escaped in the env file with an extra dollar sign. i.e. `abc$123$.` will become `abc$$123$$.`

```bash
TRAEFIK_DOMAIN=mydomain.com
TRAEFIK_LOG_LEVEL=DEBUG
TRAEFIK_CREDENTIALS="USER:a_hashed_password"
# Our DNS API credentials. For a full list go to https://docs.traefik.io/https/acme
DO_TOKEN=DIGITAL_OCEAN_API_TOKEN
```
Finally we deploy our stack expanding our environment variables, the reason for this is that `docker stack deploy` doesn't support the `.env` file[**](https://github.com/moby/moby/issues/29133). In the near future our credentials and sensitive data should be loaded using Docker secrets.

```bash
cd traefik
env $(cat env | xargs) docker stack deploy -c - traefik
```
If everything goes well, in a few seconds we'll have traefik routing our domain and the dashboard working at `https://traefik.mydomain.com` 

### Deploying services to Swarm with Traefik annotations.

The main difference between Traefik v1 and v2 is that we no longer talk about front-ends and back-ends but rather `routers`, `services` and `middlewares` in between. When we use Traefik v2 in a Docker Swarm we also need a load balancer annotation that will tell Traefik about [where to look for our service](https://docs.traefik.io/providers/docker/#port-detection_1)

A simple micro service is **whoami** from the creators of Traefik. It returns basic information of the request to the endpoint. Let's take a look at the docker-compose file (docker stack file in swarm terms)

```yaml
version: "3.7"

services:
  whoami:
    image: "containous/whoami"
    networks:
      - internal
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=internal"
        - "traefik.http.routers.whoami.rule=Host(`whoami.mydomain.com`)"
        - "traefik.http.routers.whoami.entrypoints=https"
        - "traefik.http.routers.whoami.tls=true"
        - "traefik.http.routers.whoami.tls.certresolver=le"
        # Swarm Mode
        - "traefik.http.services.whoami.loadbalancer.server.port=80"
networks:
  internal:
    external: false
    driver: overlay
```
If we deploy this service to our swarm cluster with `docker stack deploy -c docker-compose.yml whoami`
in a few seconds  it will be available at `https://whoami.mydomain.com` thanks to our wildcard certificate and global https redirection. The same pattern can be repeated with any other service and we could use other Traefik router features on top, i.e. use directory routing and transformations etc.

Included in this project we have a Portainer stack that once deployed will be available at `https://portainer.mydomain.com` user and password are created the first time we access the URL or can be injected into the `portainer_data`volume. 

### TODO
* Prometheus metrics with Grafana dashboard for monitoring (maybe using https://github.com/stefanprodan/swarmprom)
* Load sensitive data using Docker secrets
* HA services examples(including Traefik itself)








