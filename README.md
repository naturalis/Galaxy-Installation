# Galaxy 2019 Naturalis
This repo documents the installation of **Galaxy** `v.19.01` on **Ubuntu** `18.04 LTS`on the [OpenStack](https://stack.naturalis.nl/horizon/auth/login/) cloud environment of [Naturalis](https://www.naturalis.nl)

## installation
These instructions start from the assumption that an instance running Ubuntu (in this case 18.04.0 LTS on a virtual machine consisting of 16 CPUs, 16 GB ram and 60 GB Diskspace, a.k.a. 'flavor' hpc.16c.16r.60h) has been created and can be accessed via SSH.

## Python, Conda, Users and Groups
As of this writing, Ubuntu 18 has Python 3.x installed by default, but [Galaxy](https://docs.galaxyproject.org/) still requires Python 2.7 for install 

1.Install Python v. 2.7 (user: ubuntu)
```
sudo apt-get install python
```
2.Create 'galaxy-python' folder
```
sudo mkdir /home/galaxy-python
sudo ln -s /usr/bin/python /home/galaxy-python/
```
3.Download and install Conda
```
wget https://repo.anaconda.com/archive/Anaconda3-2018.12-Linux-x86_64.sh
sudo sh Anaconda3-2018.12-Linux-x86_64.sh
```
Use `/opt/anaconda3` for the install location,
`no` for initialization of Anaconda3 in /home/ubuntu/.bashrc
and `no` for installation of Microsoft VSCode.    
4.Add python and conda to environment
```
sudo nano /etc/environment
```
prepend path with:
```
/home/galaxy-python:/opt/anaconda3/bin:
```
```
source /etc/environment
```




