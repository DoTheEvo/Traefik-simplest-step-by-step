# Traefik v2 </br>guide by examples

requirements

- have docker running somewhere
- have a domain `whateverblablabla.org`
- use cloudflare to manage DNS of the domain
- have 80/443 ports open

chapters

1. [traefik routing to docker containers](#1-traefik-routing-to-various-docker-containers)
2. [traefik routing to a local IP addresses](#2-traefik-routing-to-a-local-IP-addresses)
3. [middlewares](#3-middlewares)
4. [let's encrypt certificate html challange](#4-let's-encrypt-certificate-html-challange)
5. [let's encrypt certificate DNS challange](#5-let's-encrypt-certificate-DNS-challange-on-cloudflare)

### #1 traefik routing to various docker containers

![traefik-dashboard-pic](https://i.imgur.com/5jKHJmm.png)


- **create a new docker network** `docker network create traefik_net`.
Traefik and the containers need to be on the same network.
Compose creates one automaticly, but there is potential for fuckups later.
Better to just create own network and set it as default in every compose file.

   *extra info:* use `docker network inspect traefik_net` to see containers connected to that network

- **create traefik.yml**(traefik.toml) contains so called static traefik configuration.</br>
Dashboard is enabled in it.</br>
Entrypoints are defined, in this case one called `web` for port 80.</br>
Providers are defined , in this case docker.</br>
Since exposedbydefault is set to false, a label `"traefik.enable=true"` will be needed for containers that should be routed by traefik.</br>
This file will be passed to a docker container using bind mount, this will be done when we get to docker-compose.yml for traefik.

    `traefik.yml`
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


- **create `.env`** file that contains variables that will be available for docker-compose
as enviroment variables when running the `up` command.
This alows compose files to be moved from system to system more freely and changes are done to the .env file,
so there's smaller possibility for a fuckup.

    `.env`
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```

    *extra info:* `docker-compose config` or `docker-compose -f traefik-docker-compose.yml config`
    show how will compose look with the variables filled in.</br>
    Also getting in to the container with `docker container exec -it traefik sh`
    and then `printenv` can be useful.

- **create traefik-docker-compose.yml** file. It's a simple typical compose file.
Port 80 is mapped since we want traefik to be in charge of it and use it as entrypoint.
Port 8080 is for dashboard where traefik shows info and stats.
Mount of docker.sock is needed so it can actually do its job communicating with docker.
Mount of traefik.yml is what gives the static traefik configuration.
The default network is defined so that it can be used in other compose files.

    `traefik-docker-compose.yml`
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
    ```

- **run traefik-docker-compose.yml**</br>
`docker-compose -f traefik-docker-compose.yml up -d` will start the traefik container.

    traefik is running, you can check it at the ip:8080 where you get the dashboard 

    *extra info:*</br>
    Typicly you see guides having just single compose file called `docker-compose.yml`
    with several services/containers and then it is just `docker-compose up -d` to start it all.
    You don't even need to bother defining networks with that.
    But this time I prefer small and separate steps when learning this new shit.
    So that's why going with custom named docker-compose files.

    *extra info2:*</br>
    What you can also see in tutorials is no mention of traefik.yml(traefik.toml)
    and stuff is just passed from docker-compose file using docker's command.</br>
    like this: `command: --api.insecure=true --providers.docker`</br>
    But that way feels bit more messy than having nicely structured yml file,
    but it does give benefit of having more centralized one-place to manage stuff,
    not this dancing between various config files. 
    Also .env file variables works for compose files but not for static config files,
    so that's another benefit of maybe rather going command way.

- **add labels to containers** that you want traefik to route.</br>
Here are examples of whoami, nginx, apache, portainer.
As is obvious from the patern, all that is needed is adding few self-explanatory labels.

    `whoami-docker-compose.yml`
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

    `nginx-docker-compose.yml`
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
    ```

    `apache-docker-compose.yml`
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

    `portainer-docker-compose.yml`
    ```
    version: '3'

    services:
      portainer:
        image: portainer/portainer
        container_name: portainer
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - portainer_data:/data
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.portainer.entrypoints=web"
          - "traefik.http.routers.portainer.rule=Host(`portainer.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK

    volumes:
      portainer_data:

    ```

- **run the damn containers**</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>
    `docker-compose -f apache-docker-compose.yml up -d`</br>
    `docker-compose -f portainer-docker-compose.yml up -d`


### #2 traefik routing to a local IP addresses

![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)


When url should aim at something other than a docker container.

- **define a file provider, add required routing and service**

  This can't be defined using labels in a compose file, so a file provider is needed.
  Somewhat common is to set traefik.yml itself as a file provider.</br>
  Under providers theres a new `file` section and `traefik.yml` itself is set.</br>
  Then there is a new dynamic configuration section.</br>
  A router with a simple subdomain hostname rule. What fits that rule,
  in this case exact url `test.whateverblablabla.org`, is send to 
  a loadbalancer service which just routes it a specific IP and specific port.

    `traefik.yml`
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
        filename: "traefik.yml"

    ## DYNAMIC CONFIGURATION
    http:
      routers:
        route-to-local-ip:
          rule: "Host(`test.whateverblablabla.org`)"
          service: route-to-local-ip-service
          priority: 1000
          entryPoints:
            - web

      services:
        route-to-local-ip-service:
          loadBalancer:
            servers:
              - url: "http://10.0.19.5:80"
    ```

    Priority of the router is set to 1000, a very high value,
    beating any possible other routers,
    like one we use later for doing global http -> https redirect.
    
### #3 middlewares

Example of authentification middleware for any container.


- **create a `users_file`** containing username:passwords pairs, [htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style

    `users_file`
    ```
    me:$apr1$wNMrGf17$xlOV5D1dtvFHGBYMCSjYM.
    admin:$apr1$hwaRrKGu$hJFHy0z3KwJHovY8TGG5J/
    bastard:$apr1$bZixfVAv$VKm66D9chzt.us.lnaJWz.
    ```

- mount users_file in traefik-docker-compose.yml

    `traefik-docker-compose.yml`
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
          - "./users_file:/users_file:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- add two labels to a container to activate middleware. 
The first label creates new middleware called my-auth.
The second label gives this middleware type of basicauth and points it to users_file.
No need to mount the users_file here.

    `whoami-docker-compose.yml`
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
          - "traefik.http.routers.whoami.middlewares=my-auth"
          - "traefik.http.middlewares.my-auth.basicauth.usersfile=/users_file"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
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
          - "traefik.http.routers.nginx.middlewares=my-auth"
          - "traefik.http.middlewares.my-auth.basicauth.usersfile=/users_file"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

  Now a valid login/password is needed before allowing access to these services


### #4 let's encrypt certificate, html challange

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a file on the server containing cryptographic key
   that allows encrypted communication and can be also used to confirm the identity of the server</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiatie certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip addresse</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts at a specific place on the server.
  LE then asks DNS internet servers for `whateverblablabla.org` and that points to some IP address.
  LE looks at that IP address through ports 80/443 for the file containing that random text.

  If it's there then this proves that whoever asked for the certificate controls both
  the server and the domain, since it showed control over DNS records.
  Certificate is given and is valid for 3 months, traefik will automaticly try to renew
  when less than 30 days is remaining.

  Now how to actually get it done.


- **create an empty acme.json file with 600 permissions**

  `touch acme.json && chmod 600 acme.json`

- **add certificate resolver to traefik.yml**</br>
using staging caServer makes LE issue a staging certificate,
it is invalid and wont give green lock, but it is good for testing.
The email is where LE sends notification about certificates ending

    `traefik.yml`
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false

    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

- **add port 443 and mount acme.json** in traefik-docker-compose.yml 

    `traefik-docker-compose.yml`
    ```
    version: "3.3"

    services:
      traefik:
        image: "traefik:v2.0"
        container_name: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers**
compared to just plain http it is literally just changing entryPoints from web to websecure
and to assigning tsl certificate resolver named lets-encr

    `whoami-docker-compose.yml`
    ```
    version: '3'

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: '3'

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **reddirect http traffic to https** by adding some labels to some compose files.</br>
http entrypoint stops working with this basic https setup.
But no point dealing with that and better to just make sure http 
is redirected to https by adding 4 labels that will create middleware 
that reddirects http traffic to https. 
There are several places that this can be declared, let's just add it to traefik compose.

    `traefik-docker-compose.yml`
    ```
    version: "3.3"

    services:
      traefik:
        image: "traefik:v2.0"
        container_name: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"
        labels:
          - "traefik.enable=true"
          - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
          - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
          - "traefik.http.routers.redirs.entrypoints=web"
          - "traefik.http.routers.redirs.middlewares=redirect-to-https"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK

    ```
  - First label enables traefik for this traefik container, but most importantly makes other labes work.</br>
  - Second label creates new middleware called `redirect-to-https` and assigns it scheme `https`.</br>
  - Third label declares rule that applise, so what url are affected, its a regex that just say everyhing.</br>
  - Forth label declares entrypoint that it is applied to.</br>
  - Fifth label assing the middleware to routers.</br>

  At least thats my understanding, I dont really truly know any of this.
  This is mess of labels, yml format is bit more readable but still just arbitary mess.
 
### #5 let's encrypt certificate DNS challange on cloudflare

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a file on the server containing cryptographic key
   that allows encrypted communication and can be also used to confirm the identity of the server</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiatie certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip addresse</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts as a new DNS txt record.
  LE then checks `whateverblablabla.org` DNS records to see if the text is there.
  
  If it's there then this proves that whoever asked for the certificate controls the domain,
  since it showed control over DNS.
  Certificate is given and is valid for 3 months. Traefik will automaticly try to renew
  when less than 30 days is remaining.

  Benefit over httpChallenge is ability to have wildcard certificates.
  These are certificates that validate for example all subdomains `*.whateverblablabla.org`</br>
  Also no ports are needed to be open.

  But traefik needs to be able to make changes to DNS records,
  for this there needs to be support on the DNS control provider site.

  Now how to actually get it done.

- **create an empty acme.json file with 600 permissions**

  `touch acme.json && chmod 600 acme.json`

- **add certificate resolver to traefik.yml** 
  
  `traefik.yml`
  ```
  api:
    insecure: true
    dashboard: true

  entryPoints:
    web:
      address: ":80"
    websecure:
      address: ":443"

  providers:
    docker:
      endpoint: "unix:///var/run/docker.sock"
      exposedByDefault: false

  certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
  ```

- to the `.env` file add `CF_API_EMAIL` - cloudflare login and `CF_API_KEY` - global api key

  `.env`
  ```
  MY_DOMAIN=whateverblablabla.org
  DEFAULT_NETWORK=traefik_net
  CF_API_EMAIL=whateverbastard@gmail.com
  CF_API_KEY=8c08c85dadb0f8f0c63efe94fx155b6ve1abc
  ```

- **add port 443 and mount acme.json** in traefik-docker-compose.yml 

    `traefik-docker-compose.yml`
    ```
    version: "3.3"

    services:
      traefik:
        image: "traefik:v2.0"
        container_name: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers** 

    `whoami-docker-compose.yml`
    ```
    version: '3'

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"
          - "traefik.http.routers.whoami.tls.domains[0].main=$MY_DOMAIN"
          - "traefik.http.routers.whoami.tls.domains[0].sans=*.$MY_DOMAIN"


    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: '3'

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"
          - "traefik.http.routers.whoami.tls.domains[0].main=$MY_DOMAIN"
          - "traefik.http.routers.whoami.tls.domains[0].sans=*.$MY_DOMAIN"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```
  
