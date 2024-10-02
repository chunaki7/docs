# Backing up Docker named volumes

### Stopping the Docker Service/Containers

To stop the Docker service on Ubuntu 22.04:\
`systemctl stop docker`

To stop containers:\
`docker stop <container name>`

To stop all containers:\
`docker stop $(docker ps -a -q)`

### Source Machine

1. `sudo su` to gain root access.
2. `cd /var/lib/docker`
3. `rsync -av --rsync-path="sudo rsync" volumes/ chunaki@192.168.0.22:/home/chunaki/volumes/`

> Before you can use `--rsync-path="sudo rsync"`, you need to set the permissions on the **destination** machine.
>
> 1. Find out the path to `rsync`: `which rsync`
> 2. Edit the `/etc/sudoers` file: `sudo visudo`
> 3. Add the line `<username> ALL=NOPASSWD:<path to rsync>`, where _username_ is the login name of the user that rsync will use to log on. That user must be able to use `sudo` {: .prompt-info }

### Target Machine

1. `sudo su`
2. `cd /var/lib/docker`
3. `cp -r /home/chunaki7/volumes/* volumes/`

### References
