# Notice, this shit is work in proggress, not fully tested

# Traefik v2 </br>simplest step-by-step how-to guide

requirements

- have docker running somewhere
- have rough idea of what the fuck docker is
- have rough idea of docker bind mounts, basicly map file from the host to a container
- have a fucking domain, like `fuckingwhateverblabla.org`
- recommend to use cloudflare for DNS
- fot https be fucking 100% sure you have port 80 open

### CHAPTER #1 </br>traefik pointing to various docker containers

- have traefik.yml file
- run traefik container using docker-compose.yml
- add labels to some containers, nginx, apache, portainer, whoami
- done

**traefik.yml**

```
entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

**traefik docker-compose.yml**
```
version: "3.3"

services:
  traefik:
    image: "traefik:v2.0"
    container_name: "traefik"
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik.yml:/traefik.yml:ro"

````

**whoami docker-compose.yml**

this one we have right at the domain, no subdomain or subfolder

```
version: '3'

services:
  whoami:
    image: "containous/whoami"
    container_name: "whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`fuckingwhateverblabla.org`)"
      - "traefik.http.routers.whoami.entrypoints=web"

networks:
  traefik_default:
    external: true
```

**nginx docker-compose.yml**

this one is at nginx.fuckingwhateverblabla.org

```
version: '3'

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    networks:
      - traefik_default
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx.entrypoints=web"
      - "traefik.http.routers.nginx.rule=Host(`nginx.fuckingwhateverblabla.org`)"

networks:
  traefik_default:
    external: true
````

**apache docker-compose.yml**

this one is at apache.fuckingwhateverblabla.org

```
version: '3'

services:
  apache:
    image: httpd:latest
    container_name: apache
    networks:
      - traefik_default
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.apache.entrypoints=web"
      - "traefik.http.routers.apache.rule=Host(`apache.fuckingwhateverblabla.org)"

networks:
  traefik_default:
    external: true
```

**portainer docker-compose.yml**

this one is at portainer.fuckingwhateverblabla.org

```
version: '3'

services:
  portainer:
    image: portainer/portainer
    container_name: portainer
    networks:
      - traefik_default
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.routers.portainer.rule=Host(`portainer.fuckingwhateverblabla.org`)"

networks:
  traefik_default:
    external: true

```

### CHAPTER #2 </br>traefik pointing not to docker containers

- use traefik.yml and in there defined file provider
- set up route to some IP or URL
- done

### CHAPTER #3 </br>let's encrypt certificate, html challange

- make some files and change their 
- add some labels
- done

### CHAPTER #4 </br>let's encrypt certificate, DNS challange

- bla
