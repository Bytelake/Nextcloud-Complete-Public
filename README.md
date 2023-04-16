# Sorta Nextcloud AIO thing

This should generally go over the steps and configuration to get a nice functioning nextcloud server. I've installed it on top of OpenMediaVault and Portainer, a Proxmox LXC running Ubuntu, and more. It works on Ubuntu, Debian, and Alpine as well, but commands have to be for Alpine.
# Tutorial
The goal of this project is to make a cloud storage server with document editing, presentation editing, etc. in a box. Like Google Workspace to some extent. Currently Onlyoffice refuses to function alongside nextcloud so it's still a work in progress. But the entire cloud storage side of things is functional.

To make server management easier, I'm using OpenMediaVault, which is based on Debian 11 as of the time of writing, which makes it pretty easy to work with. With OpenMediaVault I get a nice webui to check on the server with. 

### Networking
This is rather important if you want to access your site externally.  
You have 3 options  
1. Port forward ports 80 and 443 to your server
2. Use Cloudflare's Zero Trust Access Tunnels - this has the limitation of limiting uploads and downloads, especially for file drop links (Send a link to someone to upload files). It is the most secure and super easy. You don't even need nginx proxy manager.
3. My preferred method - Rent a $5 VPS and follow [chucklessducks'](https://github.com/chucklessducks/VPS-Wireguard-Nginx-Mailcow) tutorial to make a tunnel. It's a good balance between options 1 and 2. Essentially no upload download limitations, but no port forwarding. You can configure fail2ban and other security services on the VPS, and it also defeats NAT, since the server connects to the VPS, so WAN IP doesn't matter.  

So pick a method, and set that up once your OS is functioning. Be warned, for option 3, wireguard might screw up the internet connection for the server, causing package installs to fail and whatnot so make sure it's down until just before configuring nginx proxy manager.  

## Setup
I'll be setting up a system with a dedicated HDD for nextcloud data and then an SSD for the OS and services. I used OpenMediaVault's webUI to format and mount the HDD already, which is quite easy. So then we'll begin by symlinking the HDD to the path: /media/data-drive/  
  
```
cd /srv
ls # There will be a really long name like "dev-disk-by-uuid-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXX" - that's the HDD
ln -s /srv/(drive-name) /media/data-drive # Symlink the data drive to make referencing it easier. Saves time
mkdir /nextcloud/ # Now we make the directory where all of nextcloud and it's accompanying services will go
```
  
Then we'll update the system, and then install docker and docker compose. This'll take a few minutes.  
```
sudo apt update
sudo apt upgrade #This will take a little while
sudo apt install docker.io
sudo apt install docker-compose
```

***Now make a copy of the docker-compose.yml and change all the <INSERT_PASSWORD> to secure passwords.***  
**There are 3 passwords needed. One for Postgres, one for Nginx Proxy Manager and it's DB, and Redis.**

After that, you should be ready to deploy the stack. I personally used portainer for ease of use. To do that, just follow the instructions located one portainers website to install it.

## Deploying the Stack  

Then deploy the stack!  
### For deploying through the terminal/ssh/not portainer:  

````
cd /nextcloud/
nano docker-compose.yml
# then paste the contents of the docker-compose.yml file in this github.  
# with that done, now run:  
docker-compose up
````
### For deploying through portainer:
- Create a new stack  
- Paste in the contents of docker-compose.yml
- Remember to name it
- Hit Deploy  
  
Assuming all containers successfully deploy, you've made a lot of progress!  

### Nextcloud Initial Setup
Now we need to setup Nextcloud so it generates the config.php we'll need momentarily  
Go to: https://server_ip:4043  
Fill out all the fields. Select PostgreSQL as the database. It's details can be found in the docker-compose.yml (It's environment variables)  
Hit install and continue till you hit the dashboard or files section. Then you can close the tab or keep it open. Up to you. We will come back to it later.  

## Config Changes

Now we need to do a bunch of config changes so everything plays nice and functions smoothly. We'll start with Redis and adding the custom domain we want to use 
You need to go to Nextcloud's config.php file
````
nano /nextcloud/config/www/nextcloud/config/config.php
````
Then at the third line, replace the: "'memcache.local' => '\\OC\\Memcache\\APCu',", with:
````
  'memcache.local' => '\OC\Memcache\APCu',
  'memcache.distributed' => '\OC\Memcache\Redis',
  'redis' => [
       'host'     => 'redis',
       'port'     => 6379,
       'password' => '<REDIS PASSWORD>',
       'timeout'  => 1.5,
  ],
  'memcache.locking' => '\OC\Memcache\Redis',
  ````
***Be sure to change the password to whatever you set in the docker-compose.yml***
  
Then scroll down some until you find:
````
  'trusted_domains' =>
    array (
      0 => '192.168.8.166:4043',
      # Add yours right here. So: 1=> 'example.com',
````
That's all in config.php. Now we'll move on to php-local.ini
````
nano /nextcloud/config/php/php-local.ini
````
Add in:
````
memory_limit=2048M
post_max_size=64G
upload_max_filesize=64G
max_execution_time=3600
max_input_time=600
````
That's done. Now save and move on to default.conf:
````
nano /nextcloud/config/nginx/site-confs/default.conf
````
Modify these lines:
````
client_max_body_size 512M;
client_body_timeout 300s;  
````
So that they look like this:
````
client_max_body_size 64G;  
client_body_timeot 3600s;  
````

Now we move onto Nginx Proxy Manager.  (Yay more nginx!)  
Go to http://YOUR_IP_ADDRESS:81  
***The default login is Username: admin@example.com Password: changeme*** 
Set a new email.  
Then on the next window, set a new password.  
Then create a new proxy host.  
Add your domain name, then set the scheme to https, the forward hostname to "nextcloud" and port to 443.  
Enable "Block Common Exploits", then move to SSL tab and request a new certificate.  
Finally, go to advanced and paste in:
````
client_max_body_size 100G;
proxy_read_timeout 3600;
proxy_connect_timeout 3600;
proxy_send_timeout 3600;
proxy_buffering off;
````
Hit save, let if finish, and you're done! sort of...  
Your nextcloud site should be accessible from your domain, with a valid SSL certificate from LetsEncrypt.  
However, there's more work to do! Yay!  
All you need to do now is install the antivirus app on nextcloud, then in Administration Settings > Security,  
Set the antivirus to the ClamAV Daemon, host to clamav, and port to 3310. Hit save, and it's done. (ClamAV takes a minute or two to start up, so make sure it's fully started if it errors.)   
 
Tip! - If you use the wireguard tunnel, set the MTU to 1420 for optimal performance

If you've survived all that, you finished! Congratulations!
