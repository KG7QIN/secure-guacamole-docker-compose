# Secure Apache Guacamole (v1.5.5) 

**With client SSL certs using EasyRSA3 and docker-compose**

**Note:  Updated 20 July 2024 for Apache Guacamole v1.5.5**

This is a modified version of the original `guacamole-docker-compose` forked from the `boschkundendienst/guacamole-docker-compose` repository.

Changes include:
* Inclusion of EasyRSA3 with supporting scripts for generating a self-signed CA, server, and client certificates.
* Addition of custom "branding" under guacaomle/extensions that change the login screen (eurisko.jar)

Client certificates are generated with the included copy of EasyRSA3 using OpenSSL.  When connecting to the webserver, a client **MUST** present a valid certificate that has been previously generated wtih the server's CA.  Failure to present a valid client certificate results in an error preventing the client from accessing the login page.

This provides an additional layer of security to the Apache Guacamole gateway instance by only allowing those with a valid client cert to connect.  This is where the _**secure**_ comes from in the name above.

`regenerate-ca.sh` will setup/reset the self-signed CA and needs to be run after the prep instruction below **BUT BEFORE STARTING THE DOCKER INSTANCE**.  This is crucial as it also generates the SSL cert used by NGINX for Apache Guacamole.

`regenerate-server.sh` will regenerate the SSL server cert used by NGINX.

`generate-client.sh` will generate the client certs and package them in .p12 format for import into your browser.  This includes the root CA cert as well.

**_Please read the regenerate-ca.sh and regenerate-server.sh files, as they contain some variables that need to be set (FQDN/IP and SAN of server)_**

Except where noted, the rest of this "README" is from the original author.

Just make sure you also read the docker-compose.yml file for ports.  By default, this will use HTTPS on port 443 insead of the port listed below.
Additionally, you should clone this repo with:

<pre>
cd /opt
git clone https://github.com/KG7QIN/guacamole-docker-compose.git
</pre>

Instead of using the git clone command below.

**_NOTE: The EasyPKI scripts are setup with harded coded paths to use /opt/guacamole-docker-compose for their functions.  Changing this to another directory will require you to update the regenerate-ca.sh, regenerate-server.sh, generate-client.sh and any variables in scripts under the easyrsa3 directory_**

Once cloned, run the `prepare.sh` command ***AND THEN*** `regenerate-ca.sh` command ***BEFORE*** `docker-compose up -d`.

Make sure you generate client certs and import the .p12 file into your browser before trying to connect.  Without a valid client cert ***YOU CANNOT CONNECT***  to the Apache Guacamole Docker instance you created.

EasyRSA3 copied form the OpenVPN repo at https://github.com/OpenVPN/easy-rsa/tree/master/easyrsa3

------

This is a small documentation how to run a fully working **Apache Guacamole (incubating)** instance with docker (docker-compose). The goal of this project is to make it easy to test Guacamole.

## About Guacamole
Apache Guacamole (incubating) is a clientless remote desktop gateway. It supports standard protocols like VNC, RDP, and SSH. It is called clientless because no plugins or client software are required. Thanks to HTML5, once Guacamole is installed on a server, all you need to access your desktops is a web browser.

It supports RDP, SSH, Telnet and VNC and is the fastest HTML5 gateway I know. Checkout the projects [homepage](https://guacamole.incubator.apache.org/) for more information.

## Prerequisites
You need a working **docker** installation and **docker-compose** running on your machine.

## Quick start
Clone the GIT repository and start guacamole:

~~~bash
git clone "git clone https://github.com/KG7QIN/guacamole-docker-compose.git"   <--- updated to this repo
cd guacamole-docker-compose
./prepare.sh
docker-compose up -d
~~~

Your guacamole server should now be available at `https://ip of your server:443/`. The default username is `guacadmin` with password `guacadmin`.

## Details
To understand some details let's take a closer look at parts of the `docker-compose.yml` file:

### Networking
The following part of docker-compose.yml will create a network with name `guacnetwork_compose` in mode `bridged`.
~~~python
...
# networks
# create a network 'guacnetwork_compose' in mode 'bridged'
networks:
  guacnetwork_compose:
    driver: bridge
...
~~~

### Services
#### guacd
The following part of docker-compose.yml will create the guacd service. guacd is the heart of Guacamole which dynamically loads support for remote desktop protocols (called "client plugins") and connects them to remote desktops based on instructions received from the web application. The container will be called `guacd_compose` based on the docker image `guacamole/guacd` connected to our previously created network `guacnetwork_compose`. Additionally we map the 2 local folders `./drive` and `./record` into the container. We can use them later to map user drives and store recordings of sessions.

~~~python
...
services:
  # guacd
  guacd:
    container_name: guacd_compose
    image: guacamole/guacd
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/record:rw
...
~~~

#### PostgreSQL
The following part of docker-compose.yml will create an instance of PostgreSQL using the official docker image. This image is highly configurable using environment variables. It will for example initialize a database if an initialization script is found in the folder `/docker-entrypoint-initdb.d` within the image. Since we map the local folder `./init` inside the container as `docker-entrypoint-initdb.d` we can initialize the database for guacamole using our own script (`./init/initdb.sql`). You can read more about the details of the official postgres image [here](http://).

~~~python
...
  postgres:
    container_name: postgres_guacamole_compose
    environment:
      PGDATA: /var/lib/postgresql/data/guacamole
      POSTGRES_DB: guacamole_db
      POSTGRES_PASSWORD: ChooseYourOwnPasswordHere1234
      POSTGRES_USER: guacamole_user
    image: postgres
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:ro
    - ./data:/var/lib/postgresql/data:rw
...
~~~

#### Guacamole
The following part of docker-compose.yml will create an instance of guacamole by using the docker image `guacamole` from docker hub. It is also highly configurable using environment variables. In this setup it is configured to connect to the previously created postgres instance using a username and password and the database `guacamole_db`. Port 8080 is only exposed locally! We will attach an instance of nginx for public facing of it in the next step.

~~~python
...
  guacamole:
    container_name: guacamole_compose
    depends_on:
    - guacd
    - postgres
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_HOSTNAME: postgres
      POSTGRES_PASSWORD: ChooseYourOwnPasswordHere1234
      POSTGRES_USER: guacamole_user
    image: guacamole/guacamole
    links:
    - guacd
    networks:
      guacnetwork_compose:
    ports:
    - 8080/tcp
    restart: always
...
~~~

#### nginx
The following part of docker-compose.yml will create an instance of nginx that maps the public port 8443 to the internal port 443. The internal port 443 is then mapped to guacamole using the `./nginx.conf` and `./nginx/mysite.template` files. Also the generation of this container will create a self-signed certificate in `./nginx/ssl/` with `./nginx/ssl/self-ssl.key` and `./nginx/ssl/self.cert`.

~~~python
...
  nginx:
   container_name: nginx_guacamole_compose
   restart: always
   image: nginx
   volumes:
   - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
   - ./nginx/mysite.template:/etc/nginx/conf.d/mysite.template
   - ./nginx/ssl:/etc/nginx/ssl
   ports:
   - 8443:443
   links:
   - guacamole
   networks:
     guacnetwork_compose:
   # install openssl, create self-signed certificate and run nginx
   command: /bin/bash -c "apt-get -y update && apt-get -y install openssl && openssl req -nodes -newkey rsa:2048 -new -x509 -keyout /etc/nginx/ssl/self-ssl.key -out /etc/nginx/ssl/self.cert -subj '/C=DE/ST=BY/L=Hintertupfing/O=Dorfwirt/OU=Theke/CN=www.createyourown.domain/emailAddress=docker@createyourown.domain' && cp -f -s /etc/nginx/conf.d/mysite.template /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
...
~~~

## prepare.sh
`prepare.sh` is a small script that creates `./init/initdb.sql` by downloading the docker image `guacamole/guacamole` and start it like this:

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > ./init/initdb.sql
~~~
It creates the necessary database initialization file for postgres.

## reset.sh
To reset everything to the beginning, just run `./reset.sh`.

