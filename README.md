# Galaxy ~~2019~~ 2020 Naturalis
This repo documents the installation of **Galaxy** `v.19.01` on **Ubuntu** `18.04 LTS`on the [OpenStack](https://stack.naturalis.nl/horizon/auth/login/) cloud environment of [Naturalis](https://www.naturalis.nl)
In Januari 2020 this Galaxy server was migrated to v.19.09. One of the main reasons being [user management](https://github.com/galaxyproject/galaxy/pull/8309). This version will be the last version to support Python2.

## installation
These instructions start from the assumption that an instance running Ubuntu (in this case 18.04.0 LTS on a virtual machine consisting of 16 CPUs, 16 GB ram and 60 GB Diskspace, a.k.a. 'flavor' hpc.16c.16r.60h) has been created and can be accessed via SSH.

## Python, Conda, Users and Groups
As of this writing, Ubuntu 18 has Python 3.x installed by default, but [Galaxy](https://docs.galaxyproject.org/) still requires Python 2.7 for install. Galaxy should be run as user galaxy (not ubuntu). In the instructions below we aim to be explicit about the current user, but in general: unless the install requires `sudo` the user should be `galaxy`.

* Install Python v. 2.7 (user: **ubuntu**)
```
sudo apt-get install python
```
* Create 'galaxy-python' folder
```
sudo mkdir /home/galaxy-python
sudo ln -s /usr/bin/python /home/galaxy-python/
```
* Download and install Conda
```
wget https://repo.anaconda.com/archive/Anaconda3-2018.12-Linux-x86_64.sh
sudo sh Anaconda3-2018.12-Linux-x86_64.sh
```
Use `/opt/anaconda3` for the install location,
`no` for initialization of Anaconda3 in /home/ubuntu/.bashrc
and `no` for installation of Microsoft VSCode.  
  
* Add python and conda to `path` (system wide)
```
sudo su
(rm /etc/environment; cat | sed 's_PATH="_PATH="/home/galaxy-python:/opt/anaconda3/bin:_g' > /etc/environment) < /etc/environment
exit
source /etc/environment
```
* Create `conda_group` (for permissions outside homedir, in particular `/opt`, without sudo)
```
sudo groupadd conda_group
```
* Create **NON-root** user `galaxy`
```
sudo useradd -d /home/galaxy -m galaxy
```
set password and don't publish it on this page..
```
sudo passwd galaxy
#secret_password
```
* Login as user `galaxy` and change command prompt of login shell (user: **galaxy**)
```
su galaxy
#secret_password
chsh -s /bin/bash
exit
# login again to see if worked
```
* Add `ubuntu` and `galaxy` to `conda_group` (user: **ubuntu**)
```
for user in ubuntu galaxy; do sudo usermod -a -G conda_group "$user"; done
```
* Add `conda_group` to *conda*
```
sudo chgrp -R conda_group /opt/anaconda3/
sudo chmod -R g+rwx /opt/anaconda3/
```
* 'Downngrade' Conda to `canary` as a workaround for this [issue](https://github.com/conda/conda/issues/7267#issuecomment-420571523)
```
conda config --add channels conda-canary
conda update -n base conda
```
NOTE: conda update (user: ubuntu) resulted in a *PermissionError*. For now I used "sudo su" to perform the latter command,
but this is clearly undesired.

## manage SSH keys
(user: **ubuntu**)  
To allow additional adminstrators to log in as ubuntu via SSH, add their public SSH keys to the `authorized_keys` list.
```
nano /home/ubuntu/.ssh/authorized_keys
```
To allow login as galaxy using the *same* keys, copy the authorized_keys file between the home folders of these accounts.
```
sudo cp -r /home/ubuntu/.ssh /home/galaxy/
sudo chown -R galaxy:galaxy /home/galaxy/.ssh
```
A reason why you might want to log in as galaxy (instead of login as ubuntu and change user) is to have galaxy user permissions for winSCP or SSHFS sessions.

## PostgresQL
(user: **ubuntu**)
```
sudo apt-get install postgresql postgresql-contrib
sudo update-rc.d postgresql enable
sudo service postgresql start
sudo su - postgres
createdb galaxydb
psql galaxydb
CREATE USER galaxy WITH PASSWORD '#secret_password';
GRANT ALL PRIVILEGES ON DATABASE galaxydb to galaxy;
\q
exit
```

## Install Galaxy (first part)
(user: **galaxy**)    
Login as user `galaxy` and load the modified path
```
su - galaxy
source /etc/environment
```
Clone the [latest](https://docs.galaxyproject.org/en/master/) stable release of Galaxy (19.01 as of this writing)
```
git clone -b release_19.01 https://github.com/galaxyproject/galaxy.git
```
Change permissions of the `config` folder
```
chmod 777 /home/galaxy/galaxy/config
```
Adjust the config file (/home/galaxy/galaxy/config/galaxy.yml). Either edit in place or 
upload a customized `galaxy.yml` file. The config file holds key system information. To 
name a few, this is where to specify administrators, the location of dataset files, network 
ports, system email address and Conda configuration settings. The template `galaxy.yml` file
is called `galaxy.yml.sample`; start working from a copy.
```
cp /home/galaxy/galaxy/config/galaxy.yml.sample /home/galaxy/galaxy/config/galaxy.yml
```
Edit **galaxy.yml**:  
change listening port from `http: 127.0.0.1:8080` to `http: 0.0.0.0:8080`  
Add email adress(es) of admin(s) (*uncomment* admin_users: 'john.doe@naturalis.nl')  
  
## Start Galaxy
(user: **galaxy**)  
```
sh /home/galaxy/galaxy/run.sh --daemon
```
Check if the Galaxy server can be reached from your webbrowser, e.g. `http://###.###.###.###:8080` (use the floating ip of your instance followed by *:8080* since `nginx` has not been setup).  
## Stop Galaxy  
(user: **galaxy**)
```
sh /home/galaxy/galaxy/run.sh --stop-daemon
```
## Install Galaxy (second part)
(user: **galaxy**)  
Create toolmenu file
```
cp /home/galaxy/galaxy/config/tool_conf.xml.sample /home/galaxy/galaxy/config/tool_conf.xml
```
Create folder structure
```
mkdir -p /home/galaxy/{Tools,Log}
```
Add `conda_group` to *galaxy*  
```
chgrp -R conda_group /home/galaxy/{galaxy,Tools,Log}
chmod -R g+rwx /home/galaxy/{galaxy,Tools,Log}
```
NOTE: this is a workaround for adding `conda_group` to /home/galaxy *recursively*, because
that strategy resulted in permission errors (especially with /home/galaxy/.ssh but also with
the parent directory).

## Install Nginx
(user: **ubuntu**)  
[Nginx](http://nginx.org/en/)("engine x") is an HTTP and reverse proxy server
```
sudo apt update
sudo apt install nginx
```
Replace `/etc/nginx/nginx.conf` with [nginx.conf](https://github.com/naturalis/Galaxy-Installation/blob/master/nginx.conf)  
Change `http: 0.0.0.0:8080` to `socket: 0.0.0.0:8080` in `galaxy.yml`
Start Nginx
```
sudo ufw allow 'Nginx HTTP'
sudo ufw status
sudo systemctl start nginx
```
NOTE: to stop, start or restart **nginx**
```
sudo systemctl stop|start|restart nginx
```
[Start Galaxy](#Start-Galaxy)  
Check if the Galaxy server can be reached from your webbrowser, e.g. `http://###.###.###.###` (this should now work **without** *:8080*).  

## Open files limits, PAM configuration, reboot
(user: **ubuntu**)  
To prevent Galaxy from crashing upon reaching the maximum number of open files, [raise](https://underyx.me/articles/raising-the-maximum-number-of-file-descriptors) the "maximum number of file descriptor" limit.  
[Stop Galaxy](#Stop-Galaxy)  
Edit the limits config file (`/etc/security/limits.conf`) or 
```
sudo nano /etc/security/limits.conf
```
Add the following lines:
```
*    soft nofile 64000
*    hard nofile 64000
root soft nofile 64000
root hard nofile 64000
```
NOTE: These settings need to be checked; the default value for  "ulimit -Hn" in Ubuntu 18.04 LTS is 1048576.   
So we are actually *lowering* the limit (not -n and -Sn; these have a default value of 1024)  
  
Add setting to `common-session \ -noninteractive`
```
sudo su
echo "session required pam_limits.so" >> common-session
echo "session required pam_limits.so" >> common-session-noninteractive
exit
````
View the existig limits:
```
ulimit -n; ulimit -Hn; ulimit -Sn
```
```
sudo reboot
```
Start a new SSH session and check if the limits have been increased (run **ulimit** again).

## Attach storage volume
In [OpenStack dashboard](https://stack.naturalis.nl/horizon):  
Create volume (1000 GB; no soure, empty volume)  
Add volume to Galaxy instance (Edit volume; Manage attachments)
  
(user: **ubuntu**)  
<pre>
lsblk                                            # list block devices (check if volume, supposedly sdc, exists; check size)
sudo mkfs.ext4 -L galaxy_2019 /dev/sdc           # add ext4 file system (this will ERASE existing data if present)
sudo mkdir -p /media/GalaxyData                  # this will become the mountpoint for the new volume
</pre>
*Fstab* is a config file that lists all available disks and disk partitions, and indicates how they are to be initialized.
<pre>
sudo nano /etc/fstab                             # modify fstab and add the following line:
</pre>
<pre>
LABEL=galaxy_2019       /media/GalaxyData       ext4    defaults        0 0
</pre>
<pre>
sudo mount /media/GalaxyData                     # mount the new volume
lsblk                                            # check that the volume is mounted
</pre>
If running: [Stop Galaxy](#Stop-Galaxy)
<pre>
sudo reboot                                      # check if the volume mounts at startup
</pre>

## Use volume for Galaxy data
<pre>
sudo mkdir -p /media/GalaxyData/database/files   # databse/files is a default Galaxy storage location
sudo nano /home/galaxy/galaxy/config/galaxy.yml  # edit galaxy.yml, uncomment and adjust dataset storage filepath:
file_path: /media/GalaxyData/database/files
</pre>

## Adjust config files to enable download of larger files (>1 GB) from Galaxy
A reference for this [topic](https://docs.galaxyproject.org/en/latest/admin/nginx.html#sending-files-with-nginx)  
Adjust **nginx.conf**
```
sudo nano /etc/nginx/nginx.conf
```
Add the following to your Galaxy’s server {} block:  
(find    `# proxy all requests not matching other locations to uWSGI`)
```
        location /_x_accel_redirect/ {
            internal;
            alias /;
        }
```
Adjust **galaxy.yml**
```
sudo nano /home/galaxy/galaxy/config/galaxy.yml
```
Uncomment and substitute the `nginx_x_accel_redirect_base` line with: 
```
nginx_x_accel_redirect_base: '/_x_accel_redirect'
```

## Setup HTTPS

We have setup HTTPS with the following moving parts:

- certificates are issued by letsencrypt
- we let these be fetched automatically by certbot
- the certbot also injects these into the nginx config files
- to this end we have followed [this](https://gist.github.com/cecilemuller/a26737699a7e70a7093d4dc115915de8) approach,
  which means we made the nginx configuration more modular. **Note that the config by Marten no longer fully applies**:
  we scooped out the parts that are in the `server {}` block, and migrated those to a separate galaxy.conf file. The
  top-level nginx.conf is now the default version of the installation.

## Install Galaxy tools
(user: **galaxy**)  
[List](https://github.com/naturalis/Galaxy-Installation/blob/master/naturalis_galaxy-tool_list.md) of available tools.  
Follow the instructions on the respective tool pages.

## Migrate to v.19.09
(user: **galaxy**)
cd /home/galaxy/galaxy
git fetch
git checkout release_19.09
git pull -ff-only
sh /home/galaxy/galaxy/manage_db.sh upgrade

## Set up Cron


