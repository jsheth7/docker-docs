# docker registry basics

## Overview

This article walks you through creating a secure, password-protected local or remote installation of the docker registry (version 2+).

## Running a secured registry

This write-up is based on an article from:
http://container-solutions.com/running-secured-docker-registry-2-0/

Thanks to Container Solutions for making available the nginx proxy container that works with the standard docker registry to offer an SSL- and basic-authentication-protected Docker registry. 

## Configure connection to dev1 docker machine

Import the environment variables for the dev1 docker machine, so you can connect to it from the command line (see docker-machine basics).

	eval "$(docker-machine env dev1)"
	

## Create a host folder to hold docker registry data

Create the following host folder (e.g. on Mac OS X), to hold registry data (images, etc.):

	mkdir -p ~/Sites/docker/registry/data

## Run a docker registry 2 container

Run a docker registry 2 container, which will exist within the dev1 docker machine:

	docker run -v  ~/Sites/docker/registry/data:/tmp/registry-dev --name docker-registry -d registry:2.0
	
Note that we did not specify any port mappings (with the -p flag).
If we run docker ps, we see that this container is listening on port 5000.


	docker ps

    CONTAINER ID	PORTS		NAMES
    1655885f2299	5000/tcp	docker-registry    

## Connect to the registry container via an nginx-based proxy

We will set up an nginx container-based proxy. The nginx container will be linked to the registry container [see: container linking and port mapping]. This will allow us to access the registry only via nginx.

We will also configure nginx to handle SSL termination (decryption), access control (password protected).

## SSL certificate generation / purchase

You have two options here:

* Generate a self-signed SSL certificate
* Purchase an SSL certificate

### Self-signed certificates

Note: if you generate self-signed SSL certificate, you have to configure docker-machine to create a new docker machine which allows self-signed certificates.

e.g.:

	docker-machine create --driver virtualbox dev2 –engine-insecure-registry
	eval "$(docker-machine env dev2)"
	
### Create a directory to hold your purchased / generated certificate files:

	mkdir -p ~/Documents/registry/ssl/

### Purchased certificates

I recommend you purchase an SSL certificate for your domain through dnsimple.com. dnsimple provides DNS management as well as SSL certificates.

You can sign up through my affiliate link at:


 https://dnsimple.com/r/b04f2378006d60?account_id=34529

Once you have an account, create an A record to point from registrydev1.yourdomain.com to the ip address of the docker machine you're deploying the nginx container to (e.g. 192.168.99.100 for dev1). 

You can  find this out by running:

	docker-machine inspect dev1 | grep IPAddress
	"IPAddress": "192.168.99.100",

Then purchase a regular, single-domain SSL certificate for registrydev1.yourdomain.com, or a wildcard SSL certificate for *.yourdomain.com.

You will then be able to download the necessary files from dnsimple.
Go to "SSL Certificate" under your domain, and choose the "Install certificate" option.

Then choose Nginx, and download the following files:

Certificate Bundle (something.pem)
This .pem file contains both your primary certificate and the intermediate certificates, concatenated into a single file.

Certificate Private Key (something.key)

Rename something.pem to docker-registry.crt and something.key to docker-registry.key
and make sure to save them in:

	mkdir -p ~/Documents/registry/ssl/
	
## Create a password file

We want the registry to be password protected, so we need to create a .htpasswd file. 

	mkdir -p ~/Documents/registry/config/
	htpasswd -c ~/Documents/registry/config/.htpasswd admin
	
Enter a password when prompted.

## Run the nginx proxy container

Container Solutions has made a ready-made nginx proxy container which links to the Docker registry which we previously created.

Run the docker-registry-proxy container as follows:

    docker run -p 5043:443  \
        -e REGISTRY_HOST="docker-registry" \
        -e REGISTRY_PORT="5000" \
        -e SERVER_NAME="registrydev1.yourdomain.com" \
        --link docker-registry:docker-registry  \
        -v ~/Documents/registry/config/.htpasswd:/etc/nginx/.htpasswd:ro  \
        -v ~/Documents/registry/ssl:/etc/nginx/ssl:ro  \
        -d containersol/docker-registry-proxy

This command:

* runs the containersol/docker-registry-proxy from Docker Hub
* maps port 443 from the container to port 5043 on the host (your computer)
* instructs nginx to proxy incoming connections to port 5000 on the host "docker-registry"
* helps nginx find the host docker-registry due to the container link to the previously run "docker-registry" container
* helps nginx read in the generated htpasswd file from ~/Documents/registry/config/.htpasswd, which is mounted into the container (as read-only) at /etc/nginx/.htpasswd
* helps nginx find the necessary SSL certificates from ~/Documents/registry/ssl, which are mapped (as read-only) inside the container to /etc/nginx/ssl

## Verify nginx proxy is running

    # Run in Terminal 1:
    docker ps

You should see something like:

    CONTAINER ID    PORTS                           
    db54311a4b28    80/tcp, 0.0.0.0:5043->443/tcp

## Test your connection

Make the following curl request (which assumes a username and password both equal to admin):

	curl -I -u admin:admin https://registrydev1.yourdomain.com:5043/v2/ 
	
You should see "200 OK" in the response headers:

    HTTP/1.1 200 OK
    Server: nginx/1.7.12
    Date: Fri, 03 Jul 2015 19:37:12 GMT
    Content-Type: application/json; charset=utf-8
    Content-Length: 2
    Connection: keep-alive
    Docker-Distribution-Api-Version: registry/2.0
    Docker-Distribution-Api-Version:: registry/2.0
    
## Test pushing images to the registry

### Log into your registry

	docker login -u admin -p admin -e email@email.com registrydev1.yourdomain.com:5043

You should see something like:

	WARNING: login credentials saved in /Users/username/.docker/config.json
	Login Succeeded

### Pull the hello-world image from the Docker Hub into the dev1 docker machine:

	docker pull hello-world

### Tag the hello-world image from the dev1 docker machine so it is ready to be deployed to the registry:

	docker tag hello-world:latest registrydev1.yourdomain.com:5043/hello-secure-world:latest

### Push the hello-world image from the dev1 docker machine into the registry

	docker push registrydev1.yourdomain.com:5043/hello-secure-world:latest

## Deploying a registry to a remote docker machine (host)

You can use docker-machine to create a remote registry on a cloud provider such as AWS, or Digital Ocean, with commands similar to those described above.

## More information

* Docker Registry Manual:
https://docs.docker.com/registry/

* Docker Registry Version 2 API:
https://docs.docker.com/registry/spec/api/

* Docker Machine 0.3.0 deep dive:
https://blog.docker.com/2015/06/docker-machine-0-3-0-deep-dive/

* Docker Hub:
https://hub.docker.com/

* Using Docker (running containers, port mapping, etc.):
https://docs.docker.com/userguide/usingdocker/
    
