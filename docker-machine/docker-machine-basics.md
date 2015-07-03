# docker-machine basics

## What is docker-machine?

From the docker-machine manual:
"docker-machine lets you create Docker hosts on your computer, on cloud providers, and inside your own data center. It creates servers, installs Docker on them, then configures the Docker client to talk to them."

More information:
https://docs.docker.com/machine/

## Pre-requisites

Install docker-machine 0.3.0

    curl -L https://github.com/docker/machine/releases/download/v0.3.0/docker-machine_darwin-amd64 > /usr/local/bin/docker-machine
    chmod +x /usr/local/bin/docker-machine

## Create a local docker machine 

Create a local docker machine  using virtualbox (and the standard boot2docker iso).

    docker-machine create --driver virtualbox dev1

## Examine the environment variables for the new docker machine

    docker-machine env dev1

## Configure connection to new docker machine

Import the environment variables for the newly created docker machine, so you can connect to it from the command line.


	eval "$(docker-machine env dev1)"

## Run a sample application (nginx) in a container, on the new docker machine

### Create a document root to hold the files for nginx

	mkdir -p ~/Sites/docker/nginx/docroot

### Create a sample index HTML page

	echo "Hello and welcome to sweet goodness" > ~/Sites/docker/nginx/docroot/index.html

### Run the nginx container

Map the host folder ~/Sites/docker/nginx/docroot/ to the container folder /usr/share/nginx/html

Map the host port 8080 to port 80 in the container:

	docker run --name my-nginx -v ~/Sites/docker/nginx/docroot/:/usr/share/nginx/html:ro -p 8080:80  -d nginx

### Preview the nginx-run website

Find the IP address for dev1:

	docker-machine inspect dev1 | grep IPAddress
	"IPAddress": "192.168.99.100",
	
Go to http://192.168.99.100:8080/ in your browser, or run

	curl http://192.168.99.100:8080/

You should see: 

	Hello and welcome to sweet goodness!
	
## More Information

Docker Machine 0.3.0 deep dive:

https://blog.docker.com/2015/06/docker-machine-0-3-0-deep-dive/