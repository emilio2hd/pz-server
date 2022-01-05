# Project Zomboid Server Config
An ansible playbook to set up a [Project Zomboid](https://store.steampowered.com/app/108600/Project_Zomboid/) server.

This playbook was created base on:
- Ubuntu 20-04
- Scaleway instance
- Ansible 2.10.9
- Python 3.9
- MacOS/Linux local machine

The playbook will:
- Create a `pz-server` folder in `/home/pzuser`, which contains all the files required to run the zomboid server.
- Create a `/home/pzuser/.local/bin`, with binaries to start, stop, backup and run the server on startup.
- Create a backup repository `/home/pzuser/pz-server/backups/repository` using [restic](https://restic.readthedocs.io/en/stable/)
and a cron task to backup the folders `Zomboid/Server`, `Zomboid/db`, `Zomboid/Saves/Multiplayer/<server name>`.

# Before running the playbook
Before running this playbook, it's required a few things.

## SSH Configuration
Configure the ssh with user and indentity key (add this key to your server):
```
Host <server ip>
        HostName <server ip>
        User root
        IdentityFile ~/.ssh/id_rsa
```

Make sure you're able to access the instance using only `ssh <server ip>`.

## Create Pzuser
This playbook relies on a user called `pzuser`. Create one with:
```
adduser --disabled-password --gecos "" pzuser -q
chpasswd <<<"pzuser:<your password>"
usermod -aG sudo pzuser
```

Make sure you are able to do `sudo su pzuser` without typing password.

## Create Swap File
This is not strictly required, but some instances are created without the swap area, relying only on server memory.
In order to not have your server killing processes due to lack of memory, create a swap file with the commands:

```
# Create and use swap file
sudo fallocate -l 2G /swapfile
ls -lh /swapfile
sudo chmod 600 /swapfile
ls -lh /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show

# Making the swap file permanent
sudo cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
You can read more about it here: https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04

## Firewall
Make the server a bit more secure.
```
sudo apt-get install ufw

sudo ufw default deny             # Defining the policy, that refuses everything by default
sudo ufw default allow outgoing   # Enable outgoing traffic.

sudo ufw allow 22/tcp     # Authorize SSH
sudo ufw allow 80/tcp     # Authorize HTTP
sudo ufw allow 443/tcp    # Authorize HTTPS
sudo ufw allow 53         # Authorize DNS

# Steam and zomboid ports
sudo ufw allow 8766/udp
sudo ufw allow 8767/udp
sudo ufw allow 16261/udp

# TCP ports will have to be opened for each player slot on the server.
sudo ufw allow 16262:16272/tcp    # Server with 10 player slots

sudo ufw enable
```

More about it:
- https://www.scaleway.com/en/docs/tutorials/installation-uncomplicated-firewall/
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04-pt

## Creating ansible files
From the `inventory.example`, create a `inventory` file and change `<server ip>` to your server ip address. Ex.:
```
[pzserver]
192.168.01.20     ansible_connection=ssh
```

You can create a `config.yml` to overwrite the values on `default.config.yml`. If you wanna create a server with a different name. Like, the same server name as the current you already have.
```
# config.yml

zomboid_servername: "HellOnEarth"
zomboid_server_password: "SuperStrongPwD"
zomboid_server_admin_password: "IamTheBoss!"
```

# Running the playbook
Now that all the files are in place, you should be good to run the playbook. On your local machine, run in the terminal:
```
ansible-playbook site.yml
```

## First run
Once the playbook complete successfully, go to you server via ssh `ssh <server ip address>`, change the user to pzuser with `sudo su pzuser` and run:
```
cd ~                            # Should lead you to /home/pzuser
source .profile                 # Load the scripts in /home/pzuser/.local/bin
start-zomboid && screen -r      # Run the server and display the logs
```
You should se the dedicated server initializing and printing the logs. Once you see:
```
LOG  : General blahblah ######
Server Steam ID <bunch of numbers>
##########
```
In order to exit screen without killing the server type: `ctrl + a` followed by `ctrl + d`.
Try to not to kill the server, it lead to not saving the world properly and you losing your stuff. Always use as `pzuser`:
```
source .profile
stop-zomboid
```
You should be good to connect to the server following the instruction here [Connecting to the Server](https://pzwiki.net/wiki/Dedicated_Server#Connecting_to_the_Server).

# Migrating your save
This step is if you have a dedicated server and want to move the files to the new one.

## Save files
Go to the save folder(Zomboid/Saves/Multiplayer/<your server name>) and run the command:
```
tar -czf <your server name>.tar.gz --exclude=<your server name>.tar.gz .
```
This will compact all the files in the root of the compacted one.
Attention! <your server name> should match with the same name as in `zomboid_servername:` in default.config.yml or `config.yml`.
So, if you have:
```
# config.yml
zomboid_servername: "HellOnEarth"
```
You should compact the files like:
```
tar -czf HellOnEarth.tar.gz --exclude=HellOnEarth.tar.gz .
```

Copy the `<your server name>.tar.gz` to the folder `pz-server/files/saves`.

## Server files
Copy the files in the `Zomboid/Server` folder to the folder `pz-server/files/server`. Make sure the files are prefixed with `zomboid_servername` value.
So, if you have:
```
# config.yml
zomboid_servername: "HellOnEarth"
```
You should have the files:
```
pz-server/files/server
├── HellOnEarth.ini
├── HellOnEarth_SandboxVars.lua
└── HellOnEarth_spawnregions.lua
```

Now, with files in place, run:
```
ansible-playbook site.yml --tags "migrate-files"
```

Run the server again and create the user with the same name as the old server and you should get the world and the characters in the new server.


Have fun!

# Links
https://linuxize.com/post/how-to-use-linux-screen/
https://www.digitalocean.com/community/tutorials/how-to-back-up-data-to-an-object-storage-service-with-the-restic-backup-client
https://pzwiki.net/wiki/Startup_parameters
https://pzwiki.net/wiki/Dedicated_Server
