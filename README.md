# Sorta Nextcloud AIO thing

This should generally go over the nextcloud configuration to get a nice functioning nextcloud server

Nextcloud is installed with Portainer and on top of OpenMediaVault.

Several folders/directories need to be created:
  - Nextcloud's config @ /nextcloud/config
  - Postgres' config @ /nextcloud/postgres-config
  - Nextcloud's data folder - this is more complicated. If you have a dedicated data drive symlink it for ease of use at /media/(drive symlink) then create the directory     /media/(drive symlink)/main/nextcloud/data/

With those created, create the stack using the docker-compose.yml.  
Then navigate to https://server-IP:4043  
Input username and password, then hit the Storage and Database dropdown and select postgres.   
Input the details found in the docker-compose. The Database host is 'postgres'  
Then hit install and wait.  
After you've made your way to the dashboard, go back to portainer and into the console for the nextcloud container  

Add the nginx config changes to nginx in the container, also navigate to the nginx proxy manager webui and set up the proxy host.  

Also in the nextcloud container, paste the php-local.ini contents in this project into the same named file in /config/php/  

Finally, navigate to /config/wwww/nextcloud/config/config.php and add your domain under the:  
  0 => local-ip-address:4043  
  so that it looks like:  
  1 => domainname.tld  
  
That's it! Set up and ideally experience no problems or timeout errors!   

