---
title: 'How to create a systemd service'
published: 2026-01-05
draft: false
description: 'Some steps on how to create a systemd service, using a minecraft server as example'
tags: ['random']
---

Hi, since some time I have rented a VPS to play with backend development
and to deploy some simple apps.

The coding was fine. I used Rust + Axum to make the backend, Postgresql as
my main database and Vue + Vite to make the fronted. Something simple.

The thing is that everything work nice when I run the code in ssh with
cargo run. But this makes that my app closes when I finish the ssh session.

This is obvious and the desired effect, to prevent open and open a lot of
process. Its like closes the terminal window.

But I don't want to just open a ssh session every time I want to open
the app. Instead make the server open when the machine starts and keep
running in background.

As far as I know, there are two approach of this: Docker and systemd. I know that Docker is the common approach, but I feel is overpowered for what I
wanted. So I used systemd instead. Maybe when I have a really big project
with a lot of steps I will use Docker instead.

Well, the next steps are what I usually do to create a new user and set the
permissions. It's just a novice approach that works, and probably is not the
best way to do it in production environment. To exemplify the process I will
create a service that runs a Minecraft server.

First, I created a new group for the service users. This way I don't need to
handle with permissions.

```sh
sudo groupadd services
```

Everyone in this group will have access to the services folders. This is due
we will limit the persons that can open the folder in which the service run.
The next step is add my user to the services group. Just to note, a logout is
necessary to apply the change.

```sh
sudo adduser «username» services
```

The next step is create a new user for the service with the same name as the
service with the next command.

```sh
sudo useradd --system \
  --home-dir /srv/«service name» \
  --create-home \
  --shell /usr/sbin/nologin \
  «service name»
```

This command will create a new system user with the name «service name», set
and create the home folder into `/srv/«service name»`, and set the user shell
as the binary `/usr/sbin/nologin`, so the user itself cannot login in the
machine. This way we isolate the user and his privileges.

Create a user is not a necessary step, for what I read is a good practice to
prevent any software leak, it's not a panacea, but adds a extra layer of
security. We can just put the app into the user home folder and set the user
that will run the service as the same user. Both ways works.

If we crated the service user, we can test that trying to enter in the folder
via `cd` is not possible. This is due the permissions. To set permission to
our user to read and write files in the service folder the next commands are
necessary.

```sh
sudo chown -R «service name»:services /srv/«service name»
sudo chmod -R g+rw /srv/«service name»
```

The first command add recursively the files in the service folder into the
service group and the second gives permission to read and write to anyone in
the service group. This way we can just enter and do anything in the folder.
The only problem is that every file we create in that folder will have our
user instead of the service user, so we need to run the first command again
if the file is necessary for the service itself (since the service user is
not inside of the services group).

The next step is just to put the files of our service app inside the service
folder. So we need the next command.

```sh
sudo rsync -a --progress \
  --chown=«service name»:services \
  «path of service binary or folder» /srv/«service name»
```

This command create a copy of the binary or folder that contain the files
for the service. Also set the correct `chown` and `chmod` for the files. You
can also pass the flag `--chmod` to change the read, write and execution
permissions.

In a Minecraft server, the command will look something like this.

```sh
sudo rsync -a --progress \
  --chown=minecraft:services \
  /path/of/minecrraft-server /srv/minecraft
```

The last step is create the service script, that will determine how the
program will run.

```systemd
[Unit]
Description=«description of the service»
After=network.target

[Service]
User=«service name»
WorkingDirectory=/srv/«service name»/«working folder»
ExecStart=/srv/«service name»/«path of start file»
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

This script needs to be saved in `/etc/systemd/system/` directory as
`«service name».service`.

In the case of a Minecraft server, we need to create a `.sh` file to start
the server. Is possible to just use the java directly, but in that case
is necessary to use full paths, which is a kind of verbose. Its just easier
to execute a `.sh` script with the next content.

```sh
#!/usr/bin/env bash
java -Xms512M -Xmx4G -jar server.jar
```

This start the server with min 512M and 4G of RAM. Also don't forget make
executable with `chmod +x start.sh` and send the file to `/srv/minecraft`
with rsync. So the final `minecraft.service` script will look like like
this.

```systemd
[Unit]
Description=Minecrart server service
After=network.target

[Service]
User=minecraft
WorkingDirectory=/srv/minecraft/minecraft-server
ExecStart=/srv/minecraft/minecraft-server/start.sh
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

Finally we only need to reload the systemd daemon and start the service with
the next commands.

```sh
sudo systemctl daemon-reload
sudo systemctl enable «service name».service
sudo systemctl start «service name».service
```

To check if the service is running correctly we can use `systemctl status`
and to read the logs we can use the next command.

```sh
sudo journalctl -u «service name».service
```

And if everything goes fine (which is rarely) this should be enough to
maintain a Minecraft server always open. The only problem is that we cannot
use the terminal as the server, cause `journalctl` is just read only. To
use it, is necessary to open the server in a `tmux` session, but that requires
more configurations.

Maybe in another moment will do a script that use `tmux`, but for now, this
is enough for almost all my necessities.
