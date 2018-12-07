# Docker, https-portal, KeyCloak, ShinyProxy, Shiny

We're going to use ShinyProxy with "simple" (insecure) auth available only on the LAN, and set up an instance in the cloud with KeyCloak authentication, with a KeyCloak server we'll also set up in the cloud.

I'm just going to document the fully secured setup -- the insecure setup is a subset of the work.

So far, I've only gotten KeyCloak to work with dockerized mysql. Getting it to work with a preexisting mysql database (such as the one used by OpenMRS) is a TODO.

## Overview

* [Shiny](https://shiny.rstudio.com/) is an R library for producing webapps

* [ShinyProxy](https://www.shinyproxy.io/) is an authorization and orchestration layer. It checks whether a user is authenticated, and if not, sends them to KeyCloak for authentication. It then presents an interface for launching the Shiny apps that the user is authorized to use.

* [KeyCloak](https://www.keycloak.org/) is a crazy powerful user/auth management application, which provides a management console for adding users, a login screen, and a million options.

* [MySQL](https://hub.docker.com/_/mysql/) will hold the user data, but could also be used for other things (e.g. by the Shiny apps).

* [https-portal](https://github.com/SteveLTN/https-portal) will act as a reverse proxy, securing everything over HTTPS. It runs nginx under the hood and uses letsencrypt for TLS certificates.

```
                     +----------------------------------------------------------+
                     |                                                 _____    |
                     | Docker                 +----------+            /     \   |
                     |                auth    |          |           +       +  |
                     |              +-------> | KeyCloak +---------> | MySQL |  |
                     |              |         |          |           +       +  |
                     |              |         +----------+            \_____/   |
                     |              |                                           |
+--------+           +--------------+                                           |
|        | https     |              |                                           |
| Client +---------> | https+portal |                                           |
|        |           |              |         +------------+         +-------+  |
+--------+           +--------------+         |            |         |       |  |
   |--|              |              +-------> | ShinyProxy +-------> | Shiny |  |
+--------+           |          authenticated |            |         |       |  |
+--------+           |            app data    +------------+         +-------+  |
                     |                                                          |
                     |                                                          |
                     +----------------------------------------------------------+
```


## Server Setup

Spin up a server. I'm working on Debain Stretch. I like that it doesn't have the AppArmor weirdness mentioned below.

## Docker Setup

[Install Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

* Configure daemon to be available at port 2375. Use the snippet from the ShinyProxy instructions, but use `sudo systemctl edit docker.service` to manage the override file.

* (On Ubuntu 18.04, I had to [disable AppArmor](https://forums.docker.com/t/can-not-stop-docker-container-permission-denied-error/41142/7) because otherwise Docker containers were unkillable. But this will make snaps fail to start. Good luck.)

Add your user to the docker group.

`sudo usermod -aG docker $(whoami)`

[Install Docker Compose](https://docs.docker.com/compose/install/)

Create the network that your docker stuff will operate on:

`sudo docker network create sp-net`

## ShinyProxy Initial Setup

We're going to build a docker image for ShinyProxy exactly as described [here](https://github.com/openanalytics/shinyproxy-config-examples/tree/master/02-containerized-docker-engine), except that our network is called `sp-net`, and we're going to call our image `shinyproxy` rather than `shinyproxy-example`. Don't worry about the contents of `application.yml` just yet.

## Get Everything Initially Running

You'll need two (sub)domains, one for ShinyProxy and one for KeyCloak. Mine were `sp.domain.com` and `sp-keycloak.domain.com`. Add A records from these (sub)domains to your server's IP address.

Take a look at the [KeyCloak Docker documentation](https://hub.docker.com/r/jboss/keycloak/). Get mysql up and running (note that the database name, user name, and password all matter — see note about env vars below):

`docker run --name mysql -d --net sp-net -e MYSQL_DATABASE='keycloak' -e MYSQL_USER='keycloak' -e MYSQL_PASSWORD='password' -e MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASS" mysql:5.6`

(This could probably be integrated into docker-compose as well.)

Create a docker-compose.yml like the following:

```
version: '3'

services:
  https-portal:
    image: steveltn/https-portal:1
    ports:
      - '80:80'
      - '443:443'                                                                                                         
    networks:
      - sp-net                                                                                                            
    restart: always
    environment:                                                                                                          
      DOMAINS: 'sp-keycloak.domain.com -> http://dockerhost:8010, sp.domain.com -> http://dockerhost:8020'  # change 'domain.com' to your domain
      STAGE: local  # change this to 'production' once you're sure DNS is working
  keycloak:
    image: jboss/keycloak
    ports:
      - '8010:8080'
    networks:
      - sp-net
    environment:
      PROXY_ADDRESS_FORWARDING: 'true'
  shinyproxy:
    image: shinyproxy
    ports:
      - '8020:8080'
    networks:
      - sp-net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
networks:
  sp-net:
    external: true
```

And run it with `docker-compose up -d`

This will spin up HTTPS-PORTAL exposed at ports 80 and 443 (make sure your firewall is configured such that these ports are exposed; and that they are the *only* ports exposed), ShinyProxy on localhost:8020, and KeyCloak on localhost:8010. HTTPS-PORTAL will route to either ShinyProxy or KeyCloak depending on the URL at which it is accessed.

I wasn't able to get the KeyCloak environment variables for setting database address or database password to work. We need to do so if we want to use an un-dockerized or differently-named MySQL instance (or a different database entirely), or if we want another layer of security in between the system and the data.

## KeyCloak Setup

Start by creating the initial admin user:

`docker exec $CONTAINER_ID keycloak/bin/add-user-keycloak.sh -u admin -p admin123`

KeyCloak wisely disallows admin console access from anywhere outside localhost. So, from your dev machine, open a tunnel:

`ssh -L 4000:localhost:8010 $MY_SERVER`

Navigate to localhost:4000 in your browser and sign in with the initial admin credentials from above.

There's a zillion options! Fortunately we only need a few of them.

1. First, hover over "Master" in the upper-left corner and click "Add Realm." Name your realm. I used "shinyproxy".
2. Create your first user!
    1. Click "Users" in the left sidebar. Create a user. Only Username is required.
    2. Click on your newly created user. Click the "Credentials" tab. Set a temporary password. So you can log in as this user.
3. Click "Clients" in the left sidebar.
4. Click "Create." Name your client application. I used "shinyproxy" again.
5. On the main Settings page:
    1. Turn "Authorization Enabled" ON. (not sure if strictly necessary)
    2. Add https://sp.domain.com/* to "Valid Redirect URIs" (not sure if strictly necessary)
    3. Click "Save"
6. Click the "Credentials" tab in the top tab bar. On this page:
    1. Copy the **Secret**

## Configure ShinyProxy to use KeyCloak

We're going to edit ShinyProxy's `application.yml` and rebuild the docker image. With the names I used to configure KeyCloak in the admin console earlier, my `application.yaml` ended up looking like

```
proxy:
  port: 8080
  authentication: keycloak
  useForwardHeaders: true  # not sure if necessary or not
  admin-groups: admins
  keycloak:
    realm: shinyproxy                                                     
    auth-server-url: https://sp-keycloak.domain.com/auth
    resource: shinyproxy                                                  
    credentials-secret: ****Secret**** # the secret we copied earlier
  docker:
      internal-networking: true
  specs:
  - id: 01_hello
    display-name: Hello Application
    description: Application which demonstrates the basics of a Shiny app 
    container-cmd: ["R", "-e", "shinyproxy::run_01_hello()"]
    container-image: openanalytics/shinyproxy-demo
    container-network: sp-net
  - id: 06_tabsets
    container-cmd: ["R", "-e", "shinyproxy::run_06_tabsets()"]
    container-image: openanalytics/shinyproxy-demo
    container-network: sp-net
logging:
  file:
    shinyproxy.log
```

Run `docker build . -t shinyproxy` and `docker-compose up -d`.

One last piece before we're done: execute `docker pull openanalytics/shinyproxy-demo` to pull the Shiny demo apps that ShinyProxy references in the `application.yml` above.

You should now be able to navigate to `sp.domain.com`, be redirected to `sp-keycloak.domain.com`, log in as the user you created, be redirected back to `sp.domain.com`, and launch a Shiny App.

## Shiny Setup

[Install R / RStudio](https://www.rstudio.com/products/rstudio/download/)

[Create R Shiny application](https://shiny.rstudio.com/articles/basics.html)

[Package it up](https://www.shinyproxy.io/deploying-apps/), but use `FROM [rocker/r-base:latest](https://github.com/rocker-org/rocker)`

**Hopefully this all works for you!**

