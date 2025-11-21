---
title: "From Prototype to Persistent Service: Self-hosting Suwayomi with Docker Compose on a Linux Server (Part 2)"
date: 2025-11-21
categories: ["software"]
tags: ["Podman", "Self-hosted", "Manga"]
description: "Part two on how I set up my own Suwayomi server with Podman on my local network."
draft: false
---

## The preamble

A while ago I wrote an article that explained how tired I was of switching manga websites every time one is taken down, and how hard it was to track new releases and updates. I actually used it for a while, and I encountered some issues; Although the overall experience was pleasant, there were some things I wanted to improve. So, I've decided to write this follow-up article.

## The issue

### The production server

The first thing I decided I wanted to tackle was the deployment location. It was running on my laptop, and I didn't want to keep turning the service on and off. Fortunately, I was able to spend some time this month onto setting up my home lab server, and the deployment of this manga reader service was the perfect excuse for the first experiment.

For this experiment, I decided on the following requirements:

- The Suwayomi instance will be running on port 4567, which is the default.
- No outside connections to the instance, only from local network.
- Migration from podman desktop to a docker / podman compose format, to make it portable.
- The Suwayomi image will be the stable version.

During my testing I found some issues:

- The version I was using for the container had some kind of issue with certain websites. I think it was related to some unmet dependencies in the image due an update, so I took the stable one instead.

- Comics weren't updating properly. Turns out I misconfigured some options on automatic updates, so no update was being done.


## The process

First things first, I installed a fresh copy of fedora 43 server edition and configured the whole OS permissions and ports. The process was really easy and in under an hour I had everything ready to go.

If you are running something else, such as Debian or Ubuntu (or Docker instead of Podman), the process will be mostly the same, but you'll have to adapt to your system.

The only special thing I had to do was to open the 4567 port on the firewall.

```bash
sudo firewall-cmd --add-port=4567/tcp --permanent
sudo firewall-cmd --reload
```

The reason for this rule is because we will open the same port on the container, so we can map the port 4567 of the container to the 4567 of the server. I'm not using the standard http port for the service because there will be other services on this server (for example, this blog) and they will require some different treatment.

Then, I installed podman-compose:

```bash
sudo dnf install podman-compose
```

This podman-compose package provides an interface so podman can use docker compose files. It's compatible with docker compose.

### The Compose file:

After investigating a little bit, I inspected the Kubernetes (kube) configuration file that was generated on podman desktop, and came out with the following .yml configuration file:

```yml
version: '3.8'
services:
  suwayomi:
    image: ghcr.io/suwayomi/tachidesk:stable
    command: /home/suwayomi/startup_script.sh
    ports:
      - "4567:4567"
    environment:
      - TZ=Europe/Madrid
    volumes:
      - /home/user/suwayomi/data:/home/suwayomi/.suwayomi
      - /home/user/suwayomi/manga:/manga

    restart: unless-stopped
```

As you can see, it's a very easy one. Let's dive into it a little bit:

As per requirements, I switched to the stable image. Then, when a new stable comes out, I will pull again, and restart the service. Easy updates.
The command must point to the Suwayomi startup script inside the container.

```yml
 suwayomi:
    image: ghcr.io/suwayomi/tachidesk:stable
    command: /home/suwayomi/startup_script.sh
```

For the environment, I set the timezone to Europe, so the container has the same configuration than the server. Since you can configure the frequency and times of the updates on Suwayomi config, this comes in handy. If you need some special functionality, can check the [official documentation](https://github.com/Suwayomi/Suwayomi-Server-docker/pkgs/container/tachidesk#environment-variables).

```yml
environment:
      - TZ=Europe/Madrid
```

This configuration is essential for data persistence. As explained in the previous article, we map two volumes: the data folder (/home/suwayomi/.suwayomi) stores all configuration and database files, and the manga folder (/manga) allows you to add or manage downloaded content, including your own local scans.

```yml

volumes:
    - /home/user/suwayomi/data:/home/suwayomi/.suwayomi
    - /home/user/suwayomi/manga:/manga

```

"Note: Remember to replace /home/user/ with the actual path on your server."

The last part, ensures that the container restarts automatically (for example, if the server goes down and restarts)

```yml

restart: unless-stopped

```

## Wrap up

And that's it! After a short investigation, I successfully deployed to my homelab!