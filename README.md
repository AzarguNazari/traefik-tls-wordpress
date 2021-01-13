# Traefik Proxy

Traefik is a HTTP proxy and load balancer that can be used for deploying microservices easily. Traek is intengrateable with on-prem or cloud infrastructure such as Docker, Docker Swarm, Kubernetes, Marathon, Consul, Etcd, Rancher, Amazon ECS, and etc. Traefik can be configured automatically and dynamically. While deploying containers using container orchestrator, you can easily point traefik with already components and you are ready to go.  

In microservices, proxy is an important component for having a specific gateway for all incoming request and map them to the internal containers and create a load balancing between different instance of containers.  

Here is an overall view of Trafik:
![](https://github.com/traefik/traefik/raw/master/docs/content/assets/img/traefik-architecture.png)

## Trafik Features:
- Continious updates its configuration (No starts)
- Support multiple load balancing algorithm
- Provides HTTPs to your microservices by leveraging Let's Encrypt (wildcard certificates support)
- Circuit breaker, retry
- Clean web UI
- Supports for Websocket, HTTP/2, and GRPC 
- Provides metrics (Rest, Prometheus, Datadog, Statsd, InfluxDB)
- Fast
- Exposes a Rest API
- Packaged as a single binary file as well as avialable by docker image.


### Here is an example of Traefik Reverse Proxy
```
version: "3.3"

services:
  ################################################
  ####        Traefik Proxy Setup           #####
  ###############################################
  traefik:
    image: traefik:v2.0
    restart: always
    container_name: traefik
    ports:
      - "80:80" # <== http
      - "8080:8080" # <== :8080 is where the dashboard runs on
      - "443:443" # <== https
    command:
      #### These are the CLI commands that will configure Traefik and tell it how to work! ####
      ## API Settings - https://docs.traefik.io/operations/api/, endpoints - https://docs.traefik.io/operations/api/#endpoints ##
      - --api.insecure=true # <== Enabling insecure api, NOT RECOMMENDED FOR PRODUCTION
      - --api.dashboard=true # <== Enabling the dashboard to view services, middlewares, routers, etc...
      - --api.debug=true # <== Enabling additional endpoints for debugging and profiling
      ## Log Settings (options: ERROR, DEBUG, PANIC, FATAL, WARN, INFO) - https://docs.traefik.io/observability/logs/ ##
      - --log.level=DEBUG # <== Setting the level of the logs from traefik
      ## Provider Settings - https://docs.traefik.io/providers/docker/#provider-configuration ##
      - --providers.docker=true # <== Enabling docker as the provider for traefik
      - --providers.docker.exposedbydefault=false # <== Don't expose every container to traefik, only expose enabled ones
      - --providers.file.filename=/dynamic.yaml # <== Referring to a dynamic configuration file
      - --providers.docker.network=web # <== Operate on the docker network named web
      ## Entrypoints Settings - https://docs.traefik.io/routing/entrypoints/#configuration ##
      - --entrypoints.web.address=:80 # <== Defining an entrypoint for port :80 named web
      - --entrypoints.web-secured.address=:443 # <== Defining an entrypoint for https on port :443 named web-secured
      ## Certificate Settings (Let's Encrypt) -  https://docs.traefik.io/https/acme/#configuration-examples ##
      - --certificatesresolvers.mytlschallenge.acme.tlschallenge=true # <== Enable TLS-ALPN-01 to generate and renew ACME certs
      - --certificatesresolvers.mytlschallenge.acme.email=nazariazargul@gmail.com # <== Setting email for certs
      - --certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json # <== Defining acme file to store cert information
    volumes:
      - ./letsencrypt:/letsencrypt # <== Volume for certs (TLS)
      - /var/run/docker.sock:/var/run/docker.sock # <== Volume for docker admin
      - ./config/dynamic.yaml:/dynamic.yaml # <== Volume for dynamic conf file, **ref: line 27
    labels:
      #### Labels define the behavior and rules of the traefik proxy for this container ####
      - "traefik.enable=true" # <== Enable traefik on itself to view dashboard and assign subdomain to view it
      - "traefik.http.routers.api.rule=Host(`monitor.example.com`)" # <== Setting the domain for the dashboard
      - "traefik.http.routers.api.service=api@internal" # <== Enabling the api to be a service to access

  ################################################
  ####         Site Setup Container         #####
  ##############################################
  wordpress: # <== we aren't going to open :80 here because traefik is going to serve this on entrypoint 'web'
    ## :80 is already exposed from within the container ##
    image: wordpress
    restart: always
    container_name: wp
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html
    labels:
      #### Labels define the behavior and rules of the traefik proxy for this container ####
      - "traefik.enable=true" # <== Enable traefik to proxy this container
      - "traefik.http.routers.nginx-web.rule=Host(`example.com`)" # <== Your Domain Name goes here for the http rule
      - "traefik.http.routers.nginx-web.entrypoints=web" # <== Defining the entrypoint for http, **ref: line 30
      - "traefik.http.routers.nginx-web.middlewares=redirect@file" # <== This is a middleware to redirect to https
      - "traefik.http.routers.nginx-secured.rule=Host(`example.com`)" # <== Your Domain Name for the https rule
      - "traefik.http.routers.nginx-secured.entrypoints=web-secured" # <== Defining entrypoint for https, **ref: line 31
      - "traefik.http.routers.nginx-secured.tls.certresolver=mytlschallenge" # <== Defining certsresolvers for https

  ################################################
  ####     DB Container not on traefik      #####
  ##############################################
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql
    
volumes:
  wordpress:
  db:
```

For the redirect file (https scheme configuration). you can have it as `config/dynamic.yaml`:
```
http:
  middlewares:
    redirect:
      redirectScheme:
        scheme: https
``
