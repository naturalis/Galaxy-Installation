# Galaxy 2019 Naturalis
This repo documents the installation of **Galaxy** `v.19.01` on **Ubuntu** `18.04 LTS`on the [OpenStack](https://stack.naturalis.nl/horizon/auth/login/) cloud environment of [Naturalis](https://www.naturalis.nl)

## installation
These instructions start from the assumption that an instance running Ubuntu (in this case 18.04.0 LTS on a virtual machine consisting of 16 CPUs, 16 GB ram and 60 GB Diskspace, a.k.a. 'flavor' hpc.16c.16r.60h) has been created and can be accessed via SSH.

## Python, Conda, Users and Groups
As of this writing, Ubuntu 18 has Python 3.x installed by default, but [Galaxy](https://docs.galaxyproject.org/) still requires Python 2.7 for install. Galaxy should be run as user galaxy (not ubuntu). In the instructions below we aim to be explicit about the current user, but in general: unless the install requires `sudo` the user should be `galaxy`.

**1.** Install Python v. 2.7 (user: **ubuntu**)
```
sudo apt-get install python
```
**2.** Create 'galaxy-python' folder
```
sudo mkdir /home/galaxy-python
sudo ln -s /usr/bin/python /home/galaxy-python/
```
**3.** Download and install Conda
```
wget https://repo.anaconda.com/archive/Anaconda3-2018.12-Linux-x86_64.sh
sudo sh Anaconda3-2018.12-Linux-x86_64.sh
```
Use `/opt/anaconda3` for the install location,
`no` for initialization of Anaconda3 in /home/ubuntu/.bashrc
and `no` for installation of Microsoft VSCode.  
  
**4.** Add python and conda to `path` (system wide)
```
sudo su
(rm /etc/environment; cat | sed 's_PATH="_PATH="/home/galaxy-python:/opt/anaconda3/bin:_g' > /etc/environment) < /etc/environment
exit
source /etc/environment
```
**5.** Create `conda_group` (for permissions outside homedir, in particular `/opt`, without sudo)
```
sudo groupadd conda_group
```
**6.** Create **NON-root** user `galaxy`
```
sudo useradd -d /home/galaxy -m galaxy
```
set password and don't publish it on this page..
```
sudo passwd galaxy
#secret_password
```
**7.** Login as user `galaxy` and change command prompt of login shell (user: **galaxy**)
```
su - galaxy
#secret_password
chsh -s /bin/bash
exit
# login again to see if worked
```
**8.** Add `ubuntu` and `galaxy` to `conda_group` (user: **ubuntu**)
```
for user in ubuntu galaxy; do sudo usermod -a -G conda_group "$user"; done
```
**9.** Add `conda_group` to *conda*
```
sudo chgrp -R conda_group /opt/anaconda3/
sudo chmod -R g+rwx /opt/anaconda3/
```
**10.** Add `conda_group` to *galaxy*
sudo chgrp -R conda_group /home/galaxy
sudo chmod -R g+rwx /home/galaxy

**11.** 'Downngrade' Conda to `canary` as a workaround for this [issue](https://github.com/conda/conda/issues/7267#issuecomment-420571523)
```
conda config --add channels conda-canary
conda update -n base conda
```






