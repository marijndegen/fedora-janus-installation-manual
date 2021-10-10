# Installation manual version 1.0

This manual describes how to install Janus on a Fedora server. This manual assumes you have met every requirement before beginning with the installation. 

### Requirements

* A Fedora 34 server (if you are running behind a NAT router, check the useful links section, you might need some additional config)
* A domain name, YOURDOMAIN.COM will be used as an example domain in this tutorial, you should replace this text with your own domain name.
* Possibility to forward ports to the server or to disable the firewall
* SSH access
* sudo access

**Minimum requirements (2 active users)**

A fedora 34 server with 1 CPU core @ 1Ghz 2 GB RAM

**Recommended requirments (20 to 40 active users)**

A fedora 34 server with 2 CPU core @ 1Ghz 2 GB RAM

## 1. Software update, dependencies and tools

**General software update**

sudo dnf update

**Dependancies**

sudo dnf install jansson-devel.x86_64 libconfig-devel.x86_64 libnice-devel.x86_64 openssl-devel.x86_64 libsrtp-devel.x86_64 usrsctp-devel.x86_64 libmicrohttpd-devel.x86_64 libwebsockets-devel.x86_64 cmake.x86_64 nanomsg-devel.x86_64 libcurl-devel.x86_64

**Tools**

sudo dnf install glib2-devel.x86_64 zlib-devel.x86_64 gengetopt.x86_64 pkg-config autoconf automake libtool.x86_64 vim git nmap

**Conclusion**

You now have every dependency you need for Janus.

## 2. Clone git, configure, install and verify installation

We got everything we need around installing Janus fetched, let's actually download Janus itself.

**Clone the repository**

git clone [https://github.com/meetecho/Janus-gateway.git](https://github.com/meetecho/Janus-gateway.git)

cd Janus-gateway

sh autogen.sh

**Configure**

./configure --help

./configure --prefix=/opt/janus --disable-data-channels --disable-rabbitmq --disable-mqtt

**Install (can take a wile)**

make

sudo make install

sudo make configs

cd ~

**Verfify installation**

ll /opt/janus 

**Conclusion**

Janus is now installed in /opt/janus, the last command executed should give the following listing: 

- bin
- etc
- include
- lib
- share

## 3. Install Nginx

Janus on it's own does not serve static files (just it's own backend), you need Nginx to serve the frontend (html, css and js). 

**Install**

sudo dnf install nginx

sudo systemctl start nginx

sudo systemctl enable nginx

sudo systemctl status nginx

**Conclusion**

You installed Nginx, but Fedora is not ready to do anything with it, this is discussed in the next phases

## 4. Configure firewall

To make Janus and Nginx accessible by the outside world, we will configure the firewall now.

**To view ports currently in use**

sudo firewall-cmd --list-ports

sudo nmap -sT -O localhost

**Configure the firewall**

sudo firewall-cmd --add-port=80/tcp

sudo firewall-cmd --add-port=443/tcp

sudo firewall-cmd --add-port=8088/tcp

sudo firewall-cmd --add-port=8089/tcp

sudo firewall-cmd --runtime-to-permanent

**Conclusion**

View the ports again and you should see the ports numbers that weren't there before.

## 5. Configure Selinux 

Configuring Selinux is easy if you know how to do it, you can also disable it but this is NOT recommended. 

**Do it the secure way, configure Selinux**

sudo semanage port -a -t http_port_t  -p tcp 8088

sudo semanage port -a -t http_port_t  -p tcp 8089

sudo setsebool -P httpd_can_network_connect 1

sudo reboot

**Do it the unsecure way, disable Selinux altogether**

sestatus

sudo vim /etc/selinux/config (set SELINUX to permissive)

sudo reboot

sestatus 

**Conclusion**

You configured the firewall and selinux (or disabled it), you should now see the Nginx sample page when visiting the server IP adres.

## 6. Install snap, certbot and recieve certificates (let’s encrypt)

So, you don't want insecure videostream connections and most browsers are not allowing you, it is easier to configure certbot, nowadays security certificates are free anyway so make use of them.

### 6.1 Install snap

sudo dnf install snapd

sudo reboot

sudo ln -s /var/lib/snapd/snap /snap

sudo snap install core

sudo snap refresh core

### 6.2 Install certbot

sudo dnf remove certbot

sudo snap install --classic certbot

sudo ln -s /snap/bin/certbot /usr/bin/certbot

### 6.3 Recieve certificates

The following command will ask you for your e-mail address, your domain name and to accept the terms of condition.

**Run certbot**

sudo certbot certonly --nginx

**Output path's**

The command above will output two file names, save these in a text editor for later use.

> /etc/letsencrypt/live/YOURDOMAIN.COM/fullchain.pem

>  /etc/letsencrypt/live/YOURDOMAIN.COM/privkey.pem

**Test automatic renewal**

sudo certbot renew --dry-run

### 6.4 Conclusion

You installed snapd to install certbot, recieved the right certificates paths which you copied to a location and tested automatic renewal of the certificates so they keep working regardless of expire dates.

###  7. Configure webserver and reversed proxy

Nginx ships with just http on port 80 available to show you an example page, we need to adjust this to redirect http to https and setup a secure way to access Janus (by using a reversed proxy).

**Configure Nginx**

sudo vim /etc/nginx/nginx.conf

**HTTP server block**

We still want http request to be served, but redirected to a secure channel. Remove the first server block and replace it with this:

```nginx
server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        return 301 https://YOURDOMAIN.COM;
}
```

**HTTPS server block**

The second server block is what will serve the static files of Janus (the frontend). Currently the server block is commented out, delete it and add this instead (Don't forget to replace YOURDOMAIN.COM with your domain):

```nginx
server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  _;
        index index.html;
        root /opt/janus/share/janus/demos/;
        autoindex on;

        ssl_certificate "/etc/letsencrypt/live/YOURDOMAIN.COM/fullchain.pem";
        ssl_certificate_key "/etc/letsencrypt/live/YOURDOMAIN.COM/privkey.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
}
```

**Serving Janus securely (reversed proxy)**

To make Janus available in a secure way the browser accept, we will create a new port listener and redirect all traffic to port 8088 internally (again don't forget to replace YOURDOMAIN.COM with your domain):

 ```nginx
server {
        listen       8089 ssl;
        listen       [::]:8089 ssl;
        server_name  _;
        root /opt/janus/share/janus;

        ssl_certificate "/etc/letsencrypt/live/YOURDOMAIN.COM/fullchain.pem";
        ssl_certificate_key "/etc/letsencrypt/live/YOURDOMAIN.COM/privkey.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;


    location ^~ / {
        proxy_pass http://127.0.0.1:8088;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_buffering off;
    }
}
 ```

**Test Nginx configuration**

sudo nginx -t -c /etc/nginx/nginx.conf

**Restart Nginx**

sudo systemctl restart nginx

**Conclusion**

When visiting the domain, you should now see HTTPS enabled, Janus itself is not working yet though.

## 7. Make Janus accessible

In this chapter we will create a deamon for Janus and add it to the user path so it can be executed in an easy way.

## 7.1 creating a daemon for Janus

Let's make Janus accessible by systemctl, for easy starting, stopping and enable it as a service.  

**Create a service file and add the content**

sudo vim /usr/lib/systemd/system/janus.service

```
[Unit]
Description=Startup for Janus

[Service]
WorkingDirectory=/opt/janus/
Type=forking
ExecStart=/opt/janus/bin/janus --daemon --log-file=/var/log/janus.log
KillMode=process

[Install]
WantedBy=multi-user.target
```

**Start Janus and make sure it stays started**

sudo systemctl status janus

sudo systemctl start janus

sudo systemctl enable janus

**Conclusion**

When rebooting the server, Janus will now start automatically. You should now be able to run the demo's as Janus is running in the background. 


## 7.2 (recommended but not required) Add Janus to path for easy debugging

Open up the bashrc file and add the export line to the very end of the file.

**Open bashrc**

sudo vim ~/.bashrc

**Add export rule to the end of the file**

```
export PATH="/opt/janus/bin/:$PATH"
```

**Refresh the path (so no reboot required)**

source ~/.bashrc

##  8. Conclusion

That is it, you just installed Janus! Read the documentation of Janus, if any questions pop up regarding the installation process you just followed, please create an issue on my github page.  

All configuration files are available at /opt/janus/etc/janus/ , you can read through them and adjust them for your ideal situation. The official Janus documentation is available at https://janus.conf.meetecho.com/

### Useful links

Running Janus client locally and server behind NAT router.


<table>
  <tr>
   <td>Description
   </td>
   <td>link
   </td>
  </tr>
  <tr>
   <td>Running Janus client locally and server behind NAT router.
   </td>
   <td><a href="https://github.com/meetecho/Janus-gateway/issues/1354">https://github.com/meetecho/Janus-gateway/issues/1354</a>
   </td>
  </tr>
</table>



## Connection problems

These topics can be the cause of connection problems:

* Extensive server load (or client load)
* Other processes on the server or on the client device can cause videostream loss.
* Not enough hardware to facilitate the calls, read the requirements
* An unstable connection.
