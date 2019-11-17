# Traefik v2 </br>guide by examples

requirements

- have rough idea wtf is docker
- have docker running somewhere
- have a domain `whateverblablabla.org`
- use cloudflare to manage DNS of `whateverblablabla.org`
- open ports for https challange

### CHAPTER #1 </br>traefik pointing to various docker containers

- **create a new docker network** `docker network create traefik_net`.
Traefik and the containers need to be on the same network.
Compose creates one automaticly, but there is potential for fuckups later.
Better to just create own network and set it as default in every compose file.

   *extra info:* use `docker network inspect traefik_net` to see containers connected to that network

- **create traefik.yml**(traefik.toml) contains so called static traefik configuration.</br>
In it is defined entrypoint named `web` and a endpoint is set to docker. Dashboard is enabled for http.
Since exposedbydefault is set to false, a label `"traefik.enable=true"` will be needed for containers that should be routed by traefik.</br>
This file will be passed to a docker container using bind mount, this will be done when we get to docker-compose.yml for traefik.

   **traefik.yml**
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
    ```

   later on when traefik container is running, use command `docker logs traefik` and check if there is `"Configuration loaded from file: /traefik.yml"`.
Or run command `docker inspect -f '{{ .Mounts }}' traefik` to see active mounts on a running container.
You don't want to be the moron who makes changes to traefik.yml and it does nothing because the file is not in use.


- **create .env** file that contains variables that will be available to compose when running `up` command.
This alows compose files to be moved from system to system more freely and changes are done to the .env file,
so there's smaller possibility for a fuckup.

   **.env**
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```

    *extra info:* `docker-compose config` shows how compose will look with variables filled in

- **create traefik-docker-compose.yml** file. It's a simple typical compose file.
Port 80 is mapped since it is used as an entryPoint where $MY_DOMAIN will land you.
Port 8080 is for dashboard where traefik shows info and stats.
Mount of docker.sock is needed so it can actually do its job communicating with docker.
Mount of traefik.yml is where the static configuration is.
The default network is defined so that it can be used in other compose files.

    **traefik-docker-compose.yml**
    ```
    version: "3.3"

    services:
      traefik:
        image: "traefik:v2.0"
        container_name: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ````

- **run traefik-docker-compose.yml**</br>
`docker-compose -f traefik-docker-compose.yml up -d` will start the traefik container.

    Now some explamation might be fitting about two possibilities.</br>
    Typicly you see guides having just single file called `docker-compose.yml`
    with several containers in it under services section and then just `docker-compose up -d` to start it all.
    You don't even need to bother with some networks with that.
    But I prefer small and separate steps when learning this new shit.
    So that's why going with custom named docker-compose files.

- **add labels to containers** that you want traefik to route to.</br>
Here are examples of whoami, nginx, apache, portainer.
As is obvious from the patern, all that is needed is adding few self-explanatory labels.
And then just run any of them with docker-compose, like this for whoami: `docker-compose -f whoami-docker-compose.yml up -d`

    **whoami-docker-compose.yml**

    ```
    version: '3'

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=web"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    **nginx-docker-compose.yml**

    ```
    version: '3'

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=web"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ````

    **apache-docker-compose.yml**

    ```
    version: '3'

    services:
      apache:
        image: httpd:latest
        container_name: apache
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.apache.entrypoints=web"
          - "traefik.http.routers.apache.rule=Host(`apache.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    **portainer-docker-compose.yml**

    ```
    version: '3'

    services:
      portainer:
        image: portainer/portainer
        container_name: portainer
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.portainer.entrypoints=web"
          - "traefik.http.routers.portainer.rule=Host(`portainer.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

### CHAPTER #2 </br>traefik routing to local IP addresses or wherever

If url should not aim at some docker container, a new file provider is needed.

- **create servers.yml** with router and a service.

   **servers.yml**

    ```
    http:
      routers:
        test:
          rule: "Host(`test.whateverblablabla.org`)"
          service: test
          entryPoints:
            - web

      services:
        test:
          loadBalancer:
            servers:
              - url: "http://10.0.19.5:80"
    ```

- **define a file provider in traefik.yml** pointing at the servers.yml file

   **traefik.yml**
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
      file:
        filename: "servers.yml"
    ```

- **mount servers.yml in to traefik container** by adding a line in traefik-docker-compose.yml**

    **traefik-docker-compose.yml**
    ```
    version: "3.3"

    services:
      traefik:
        image: "traefik:v2.0"
        container_name: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./servers.yml:/servers.yml:ro"


    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ````

### CHAPTER #3 </br>let's encrypt certificate, html challange

- make some files and change their 
- add some labels
- done

### CHAPTER #4 </br>let's encrypt certificate, DNS challange

- bla
