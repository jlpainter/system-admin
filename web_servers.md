# Web Server Configuration Notes

## TLJH + RStudio Server behind Apache httpd2 proxy
The following are my configuration parameters for enabling an Apache httpd2 server to act as a proxy between side by side installs of the [TLJH](https://tljh.jupyter.org/en/latest/ "The Littlest JupyterHub Server") version of JupyterHub and the RStudio Server on an Ubuntu Linux (20.0 tested) server.

This does NOT assume SSL is installed, but could be easily modified for an SSL setup as well.

### Enable the Required Apache Modules

```
  $ sudo a2enmod proxy
  $ sudo a2enmod proxy_http
  $ sudo a2enmod rewrite
  $ sudo a2enmod proxy_wstunnel
  $ sudo a2enmod headers

````

The tricky part was insuring that the web sockets are properly passed through the reverse proxy. The RewriteCond must come in the proper order before the RewriteRule is executed. This caused me untold hours of frustration until I finally got it right!

Modifications to my /etc/apache/sites-enabled/000-default.conf

```
        # preserve Host header to avoid cross-origin problems
        ProxyPreserveHost On
        RequestHeader set X-Forwarded-Proto "http"

        # Use RewriteEngine to handle websocket connection upgrades
        RewriteEngine On

        ##################################################
        # Jupyter Hub setup is working...
        ##################################################
        RewriteCond %{HTTP:Upgrade} =websocket
        RewriteRule /jhub/(.*) ws://127.0.0.1:8001/jhub/$1 [NE,P,L]

        RewriteCond %{HTTP:Upgrade} !=websocket
        RewriteRule /jhub/(.*) http://127.0.0.1:8001/jhub/$1 [NE,P,L]

        # Jupyter Hub setup
        # proxy to JupyterHub
        ProxyPass        /jhub/ http://127.0.0.1:8001/jhub/
        ProxyPassReverse /jhub/ http://127.0.0.1:8001/jhub/
        ##################################################

        ##################################################
        # RStudio
        ##################################################
        RewriteCond %{HTTP:Upgrade} =websocket
        RewriteRule /rstudio/(.*) ws://localhost:9001/$1 [P,L]

        RewriteCond %{HTTP:Upgrade} !=websocket
        RewriteRule /rstudio/(.*) http://localhost:9001/$1 [P,L]

        ProxyPass        /rstudio/ http://localhost:9001/rstudio/
        ProxyPassReverse /rstudio/ http://localhost:9001/rstudio/

        # Use preferably this (store variable values with dummy rewrite rules)
        RewriteRule . - [E=req_scheme:%{REQUEST_SCHEME}]
        RewriteRule . - [E=http_host:%{HTTP_HOST}]
        RewriteRule . - [E=req_uri:%{REQUEST_URI}]
        RequestHeader set X-RStudio-Request "%{req_scheme}e://%{http_host}e%{req_uri}e"
        ##################################################

        # Make the server secure again
        ProxyRequests Off

```        

For RStudio I have the following in my server config file:
/etc/rstudio/rserver.conf

```
   # Server Configuration File
   www-address=127.0.0.1
   www-port=9001
   www-root-path=/rstudio/

````

For TLJH, you must use the config utility to set some parameters

Add the http port

```
  sudo tljh-config set http.port 8001
  sudo tljh-config set https.port 8443
  sudo tljh-config set base_url '/jhub/'
  sudo tljh-config set bind_url 'http://127.0.0.1/jhub/'

````

Then, add a file for the nginx traefik proxy that sits in front of TLJH

/opt/tljh/config/traefik_config.d/traefik.toml

```
[entryPoints]   
   [entryPoints.http]
   address="127.0.0.1:8001"

````

Once you have created this file, you need to inform tljh to update the proxy settings.

> $ tljh-config reload proxy

Then restart all the services and you should be good to go!

> <p> $ sudo systemctl restart rstudio-server.service </p>
> <p> $ sudo systemctl restart jupyterhub.service </p>
> <p> $ sudo systemctl restart apache2.service </p>







