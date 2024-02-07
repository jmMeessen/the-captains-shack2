+++
author = ""
comments = true
date = "2016-11-22T22:00:00+01:00"
draft = false
image = "images/blog/old/logo-lime-survey.png"
menu = ""
share = true
slug = "lime-survey"
tags = ["docker"]
title = "Needed a survey. Easy peasy with Docker."

+++

***

**[TL;DR]**

Docker allowed me to quickly evaluate and run a complex setup. This article describes this experience.

***

Begin November, I gave a Docker one-day training at the Science Faculty of the Aix-Marseille University (Marseille - Luminy). Somehow, we failed to have the evaluation forms filled in at the end of the day. As the Training Dept. required these forms, we had to find an alternate solution that would work for the people that had to leave the training in a hurry. 

Somebody suggested "Lime Survey". I didn't know that tool but it seemed quite popular in the scientific community. Google Forms would certainly have made the trick but **as I am curious, I wanted to give Lime Survey a shot**.
Like Wordpress, there is a hosted offer with a free/demo option. I wasn't able to make it work. I somehow mixed-up the registration.

I quickly jumped to the "Plan B": setup my own instance. 

Lime Survey is a PHP application relying on a _mySQL_ database. Quite some work to setup on my Macbook Pro. But a good looking Docker image is available (https://github.com/crramirez/limesurvey). It took me just a minute to setup and run the application with:

{{< highlight bash "hl_lines=2">}}
docker pull crramirez/limesurvey:latest
docker run -d --name limesurvey -p 80:80 crramirez/limesurvey:latest
{{< /highlight >}}

This allowed me to explore the product and fine tune the configuration. Lime Survey is indeed a very complete product with invitation management, reply tracking, various exports and statistics.

Once I understood the product and built a useful evaluation survey, it was time to push it to the Captain's Shack so that trainees could fill it while it was still meaningful.

This went also surprisingly fast. Just a few preparation steps and updating Ansible, and I had a working instance.

1. add a new subdomain for survey requests
2. add a reverse proxy drop-in configuration file to the Nginx server
3. request a LetsEncrypt certificate for the new domain (automated with Docker)
4. update the master `docker-compose.yml` to include the Lime Survey container in the Shack's configuration. 

Note that for the "real" configuration the _mySQL_ database is stored in a data volume. It is persistent between restart and works better then when embedded. Keeping the database in the infrastructure container is an anti-pattern.

{{< highlight bash >}}
version: '2'

services:

  web:
    image: nginx:latest
    [...]

  limesurvey:
    image:
      crramirez/limesurvey:latest
    ports:
      - "8082:80"
    volumes:
      - lime_mysql:/var/lib/mysql

volumes:
  [...]
  lime_mysql:
    driver: local
{{< /highlight >}}

(see the github repository for details of the Captain's Shack Ansible setup scripts)

The setup of Lime Survey requires some manual configuration. I didn't have the time to automate this. But it is definitely something to be done. At least backup the configuration and the database content.

