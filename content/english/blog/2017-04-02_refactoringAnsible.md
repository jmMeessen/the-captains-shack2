+++
author = ""
comments = true
date = "2017-04-02T00:01:00+01:00"
draft = false
image = "images/blog/old/refactoringAnsible/refactoring.gif"
menu = ""
share = true
slug = "refactoringTheShack"
tags = ["Ansible", "docker"]
title = "Spring refactoring of the Captain's shack"

+++

***

**[TL;DR]**

*Ansible roles are powerful to turn features on or off. The site's monolithic master Docker-compose doesn't lend itself for that. This article describes the script's refactoring to solve this.*

***


The best way to prevent tampering on an Internet facing server is to rebuild it regularly. And the only efficient way to do it is the scripted way. From day one, the "Captain's Shack" server has been configured with Ansible so that the server can be regularly reinitialised. 
Each component, defined in an Ansible role, is deployed as a Docker container. To orchestrate these containers, I use a central "docker-compose" file.

{{< image src="images/blog/old/refactoringAnsible/Kimsufi.png" caption="" alt="kimsufi" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}


Initially, the docker configuration (and the docker-compose) was managed via a single docker-compose file. This design did not allow to disable a service (like Nexus or Lime Survey) by just commenting out the Ansible role. It required adapting several Ansible scripts or configuration file. Very impractical.  

When I wanted to turn off the "Lime Survey" service it made refactoring need obvious.

The main issue with enabling/disabling a service is the required change to the docker-compose file. And this in different sections of the file.  Obviously the service needs to be updated but also the volume section and, more annoying, the "web" service (Nginx) one. 

I initially tried to achieve this scripted per role Docker-compose update with the blockinfile Ansible module. It turned out to be clumsy. The module doesn't lend itself easily to properly handle yaml block insertions: the block is inserted at column 0. The workaround is to start the block with a dummy line that would set the correct offset of the rest of the block (as this line is inserted at column 0). That dummy line needs to be removed in the following step. Also annoying is that the module inserts each time a "noisy" comment before and after the inserted block. This makes the generated docker-compose file difficult to read for a human reader. Repeatedly manipulating individual line in the "web" service part is also very annoying and inefficient.

So I chose to go for the proverbial plan B.

The idea is to have several docker-compose.yml that will first be "superposed" before being processed by the docker-compose engine. 

{{< image src="/images/blog/old/refactoringAnsible/layers2.png" caption="" alt="kimsufi" height="" width="" position="center" command="fill" option="q100" class="img-fluid" title="image title"  webp="false" >}}

Each compose file defines what is needed to activate a single service. It describes the service and volume part but also the required additional configuration to the "web" service (reverse proxy).

{{< highlight yaml >}}
version: '2'

services:
  web:
    depends_on:
      - nexus

  nexus:
    image: sonatype/nexus3
    ports:
      - "8081:8081"
    volumes:
      - nexus-data:/nexus-data

volumes:
  nexus-data:
    driver: local
{{< /highlight >}}

These compose files can be assembled either with the `-f` switch of the docker-compose command line or by setting the COMPOSE_FILE environment variable with list of these files separated by a ":". I chose the second strategy because it was easier to setup correctly with Ansible.

So each role was modified to have each its own docker-compose file fragment. It is stored in the `data/docker` where most of the docker stuff is stored.

A script file (`setComposeList.sh`) is added so that the required environment variable is correctly set up. Each role adds the name of its compose file to the variable definition.

{{< highlight bash >}}
export COMPOSE_FILE=nginx-compose.yml:gogs-compose.yml:hugo-compose.yml:jenkins-compose.yml:nexus-compose.yml
{{< /highlight >}}

To package everything correctly, a set of scripts are also installed. I have a `start_web.sh`, a `stop_web.sh`, a `restart_web.sh` and a `build_and_start_web.sh` script. The latest is shown here after.

{{< highlight bash >}}
#!/usr/bin/env bash

cd /home/data/docker
. ./setComposeList.sh
docker-compose up --build -d
docker-compose ps
{{< /highlight >}}

The ansible scripts discussed here are available on [my Github repo](https://github.com/jmMeessen/the-captains-shack).
