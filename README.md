# odroid-server

> Notes on how to install things on odroid xu-4.

Hardware:
- odroid xu-4
- hard drives

Apps:
- Plex @ http://odroid.local --> https://apps.plex.tv
- Syncthing @ http://odroid.local/syncthing
- Transmission @ http://odroid.local/transmission

### System

- Flash Ubuntu minimal 18.04: https://odroid.in/ubuntu_18.04lts/, e.g. `ubuntu-18.04-4.14-minimal-odroid-xu4-20180531.img.xz` with etcher.io
- Mount external drives in `/etc/fstab`:

e.g. 
```
UUID=db0a76df-ad54-4afd-931c-8829626708ec /media/filme ext4 auto,defaults,nofail 2
UUID=fb248edb-064b-45a2-97fc-eefd35ff7c5a /media/musik ext4 auto,defaults,nofail 2
```
...where UUID is the one from `blkid /dev/sda1` or `blkid /dev/sdb1`

### SSH

Add user then:

On local: `~/.ssh/config`:
```
Host odroid
    HostName 192.168.X.X
    Port 22
    User usernameX
```
-> `ssh odroid`

On server: `/etc/ssh/sshd_config`:
```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
```
`service ssh restart`

### Plex

- Install Plex: https://dev2day.de/plex-media-server-arm/ for armv7 - see `uname -i`
- Enable remote access - set manual port to `80`

### syncthing

- Install syncthing: https://apt.syncthing.net/
- systemd as user: https://github.com/syncthing/syncthing/blob/master/etc/linux-systemd/user/syncthing.service
- edit .config/syncthing/config.xml to allow `0.0.0.0`
- no HTTPS in GUI (-> settings)

### Transmission

- Install transmission: `apt install transmission-daemon`
- edit config `"rpc-whitelist": "127.0.0.1",` in `/var/lib/transmission-daemon/.config/transmission-daemon/settings.json` and `incomplete` and `downloads` location
- `chown -R debian-transmission: /media/filme/incomplete`
- `chown -R debian-transmission: /media/filme/downloads`

### S3 backup

#### hard drive
- `apt install s3cmd`
- `/root/.s3cfg` -->

```
[default]
host_base = sos-ch-dk-2.exo.io
host_bucket = [bucket-name].sos-ch-dk-2.exo.io
access_key = [API-KEY]
secret_key = [SECRET-KEY]
use_https = True
```

add cron:

`0	8	*	*	*	s3cmd sync /media/musik/music s3://[bucket-name] >/dev/null`


#### home folder

`/root/backup-home.sh` -->

```bash
#!/bin/bash
tar -cvpzf /root/backup.tar.gz --exclude=/root/backup.tar.gz --one-file-system /home/david
s3cmd put /root/backup.tar.gz s3://[bucket-name-2] >/dev/null
```
add cron:
`@daily bash /root/backup-home.sh > /dev/null`



### Network & reverse proxy

- `apt install avahi-daemon` so that server is locally reachable at `http://hostname.local`
- `apt install nginx`
- `cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak`
- `vim /etc/nginx/nginx.conf` -->

```
worker_processes auto;
error_log /var/log/nginx/error.log;
events {

	worker_connections 1024;
}

http {

	#Upstream to Plex
	upstream plex_backend {
		server 127.0.0.1:32400;
		keepalive 32;
	}
	
	server {

		listen 80;
		server_name odroid.local;

		#transmission config
		location /transmission/ {
			proxy_pass_header X-Transmission-Session-Id;
			proxy_set_header  X-Forwarded-Host $host;
			proxy_set_header  X-Forwarded-Server $host;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass        http://127.0.0.1:9091/transmission/web/;
		}
		location /rpc {
			proxy_pass http://127.0.0.1:9091/transmission/rpc;
		}
    
                # syncthing config
		location /syncthing/ {
			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header X-Forwarded-Proto $scheme;

			proxy_pass       http://localhost:8384/;

			proxy_read_timeout 600s;
			proxy_send_timeout 600s;
		}
                
                # plex config
		location / {
			proxy_pass       http://plex_backend;
		}
	}
}
```

### ufw

- `apt install ufw`
- `ufw allow ssh`
- `ufw allow www`

Open another ssh session.

- `ufw enable`

Test ssh login.




