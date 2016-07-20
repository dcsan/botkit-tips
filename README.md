# Setting up a SSL server for botkit

## Preamble
These are my notes taken as I went through the process. I'll probably refer to this and edit it next time I do a new setup.
The flow below is a little confused, so I'd appreciate any comments or PRs from others going through the process, then we can clean this up and add it to the botkit repo README.

I followed the DigitalOcean guide here, but have added more info that maybe helpful.

[DigitalOcean guide](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04)

replace `EXAMPLE.COM` with the actual domain you're using below!

## Why do we need https?

Facebook requires your server to be running on https, so this means you need to setup an SSL certificate. There are a couple of ways to do this.

[Let's Encrypt](https://letsencrypt.org) gives free SSL certs.
Once you have the certificate you have to serve it up.

If you're like me, you'd prefer to get back to writing code and avoid devops stuff.
So you *could* try to get nodeJS to serve up the certificate, using [this NPM package](https://github.com/Daplie/letsencrypt-express).
But I [had problems with this](https://github.com/Daplie/letsencrypt-express/issues/58)

So the way most people do this is setting up a reverse-proxy with NgInx.
Another benefit of setting up NgInx is you just have to do it once and then you can put multiple node/botkit apps on the same server.

So let's try that :)

These instructions are for Ubuntu 14, running on a digital ocean setup.



## Spin up a machine
If you don't have an instance, go spin one up.
For this tutorial I'm using Digital Ocean and Ubuntu 14.04
Wait til it's running and you have an IP address for the machine. Note it down!

## Buy a domain
You'll need a domain to use since a certificate can't work with a raw IP address.
Note - with letsencrypt you can't use a subdomain (yet). So you'll need to probably buy a new domain.

## Edit the DNS
Create an "A record" pointing the domain at the IP number of the new machine above
Do that on the hosting provider managing your DNS.

Now, you have to wait for that domain to 'propagate' - reach your machine before you can get the certificate.
This could take 30 mins or longer. So let's do some other stuff.
You can check this using `dig YOURDOMAIN` and see if it's pointing to your new IP# yet.
Be aware that sometimes DNS propagation is uneven - so even if you see the right IP on your local machine, depending on the DNS servers your remote machine is using, it may not be updated there yet.
If you really want to speed things up, I think you can set your server to use the DNS servers of your DNS host itself... (inception!)

## install NgInx
On the new machine:

```
sudo apt-get update
sudo apt-get install nginx
```

check its running/restarting:

    sudo service nginx restart

then try accessing your server from a browser just using http.
You can try this just using the IP address, to see if nginx itself is running.
Then use the domain you have to see if the DNS has propagated yet.
You should see the `Welcome to nginx!` message.



## Get a certificate
Go to Certbot. At the time of writing there was no recipe for Ubuntu & NgInx
https://certbot.eff.org/#ubuntutrusty-nginx

So we have to get it ourselves using Wget

```
    wget https://dl.eff.org/certbot-auto
    chmod a+x certbot-auto
```

run the certbot script to validate your domain and get a certificate

```
  chmod a+x certbot-auto
  ./certbot-auto certonly --standalone -d EXAMPLE.COM -d www.EXAMPLE.COM
```
Get a certificate.
When the app opens, enter the name of your domain and your email etc.
Manually get a certificate using the standalone server.
This should shortcut having to setup NgInx to serve the certificate.

## Set the auto renewal
check the renewal script is working

```
    ./certbot-auto renew
```

if output seems good, add a crontask to run once a day and check if renewal is needed.

```
    crontab -e
```
and then in the crontab add this:

    0 14 * * * /root/certbot-auto renew --quiet --no-self-upgrade

This will set the task to run at 14:00 every day and check if the cert needs renewing.
There are various ways to debug this but at the very least come back a few days later and check there are no cron task errors.


## enable NgInx to serve up the certificate
Make sure webroot is available in nginx config

    sudo vim /etc/nginx/sites-available/default

Inside the server block, add this location block:

```
location ~ /.well-known {
  allow all;
}
```

Also look up the document root.  
The default is `root /usr/share/nginx/html;`

Restart nginx

    sudo service nginx reload

You should see an OK. So this means NgInx can now serve the certificate.
At this point you should be able to access the NgInx page via https too.


## edit nginx config

    vim /etc/nginx/sites-available/default

## modify server block

```  
        ### comment these two lines
        #listen 80 default_server;
        #listen [::]:80 default_server ipv6only=on;

        ### add these four lines
        listen 443 ssl;
        server_name EXAMPLE.COM www.EXAMPLE.COM;
        ssl_certificate /etc/letsencrypt/live/EXAMPLE.COM/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/EXAMPLE.COM/privkey.pem;
```

if you want to redirect http to https for all requests, you can also add this:

```
server {
    listen 80;
    server_name EXAMPLE.COM www.EXAMPLE.COM;
    return 301 https://$host$request_uri;
}
```
restart it, and check it's ok

```
  sudo service nginx restart
```

If your nginx config has problems or shows `fail` on restart, you can use this command to debug it:

```
nginx -c /etc/nginx/sites-available/default -t
```
[More on NgInx](https://www.nginx.com/resources/wiki/start/#pre-canned-configurations)


You can check your SSL status here

https://www.ssllabs.com/ssltest/analyze.html?d=EXAMPLE.COM&latest


## More production environment setup

For a production machine I usually use

- `n` to manage node versions (found it more reliable than `nvm`)
- pm2 to keep server processes running


```
    sudo npm install -g n
    n latest  # or whatever version of node you want
    # pm2
    sudo npm install pm2 -g
```

then lets make a sample node app
```
  mkdir -p ~/www/hello
  vim ~/www/hello/app.js
```
and paste this code into the file

```js
// replace this with your servers IP address!
var SERVER_IP="XXX.XXX.XXX.XXX";

var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(8080, SERVER_IP);
console.log('Server running at ' + SERVER_IP + ':8080/');
```

save it, and run it with `node app.js`
check it runs by accessing from a browser on its actual port
http://XXX.XXX.XXX.XXX:8080/

start it with pm2

    pm2 start app.js

OK you should now see your app running on the https port too.
https://EXAMPLE.COM/

Woot! All Done!




# Just the script

Below I'm trying to make a more abbreviated version with just the script and no explanations.

```
    # get nginx
    sudo apt-get update
    sudo apt-get install nginx
    # check it restarts ok
    sudo service nginx restart

    # manually edit the nginx config (see above)
    vim /etc/nginx/sites-available/default
```

add this to ssl block

```
        location ~ /.well-known {
                allow all;
        }
```

get certificate

```
    sudo service nginx restart   # check it works

    # get certbot
    sudo apt-get -y install git bc
    sudo git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt

    # get certificate
    # change webroot if you have a custom setting
    # change EXAMPLE.COM for your domain
    cd /opt/letsencrypt
    ./letsencrypt-auto certonly -a webroot --webroot-path=/usr/share/nginx/html -d EXAMPLE.COM

    # bit hacky to add the update cron
    # also this will depend where you've downloaded the certbot-auto script to
    # if you were logged in as root on your new DO box, this will work
    # but you should probably create a user account for this :)
    echo "0 14 * * * /root/certbot-auto renew --quiet --no-self-upgrade" > certcron
    crontab certcron
    rm certcron
    # list the crontab - you should see the command above
    crontab -l

```

then add all this stuff to the nginx config within the server block

```
    listen 443 ssl;
    server_name EXAMPLE.COM www.EXAMPLE.COM;
    ssl_certificate /etc/letsencrypt/live/EXAMPLE.COM/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/EXAMPLE.COM/privkey.pem;

```
and add a new server block (replace EXAMPLE.COM)

```
server {
    listen 80;
    server_name EXAMPLE.COM www.EXAMPLE.COM;
    return 301 https://$host$request_uri;
}
```

then reload

    sudo service nginx reload



# Other Notes

Command line to setup the cert

```
    cd /opt/letsencrypt
    ./letsencrypt-auto certonly -a webroot --webroot-path=/usr/share/nginx/html -d EXAMPLE.COM -d www.EXAMPLE.COM

    # manually get a certificate
    sudo service nginx stop   # just in case you had it installed
    ./certbot-auto certonly --standalone -d EXAMPLE.COM -d www.EXAMPLE.COM

```



# References
Great guide with more details
[DigitalOcean guide](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04)

The DO guide includes other docs on how to harden your crypto, using 2048 bit certs.

    sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
