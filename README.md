# deploy-ssr-react-app-to-linode

This is manual instruction how to deploy server side rendering react app to linode on apache server. It is not the best solution because ideally you want to automate this using circle ci , travis or other tools. Nevertheless this guide will help you to get your site running, then you can improve it. 

Step 1.

Sign up to linode service, create linode (their isntructions are very good) https://www.linode.com/docs/getting-started/

Follow all the steps: 
- connect to your linode via ssh (just as instructions says)

#### Install software
- I picked ubuntu so I followed: ```apt-get update && apt-get upgrade```

#### Set hostname
- hostname can be any word which is rememberable, which at the end it will look like something like that: ```anita@some_hostname_name``` - anita is your limited user name - you will set it up in a bit.

```hostnamectl set-hostname some_hostname_name```

#### Update your hostname files /etc/hosts
- update your hosts files, once you logged in to your linode using ssh, navigate to ```nano /etc/hosts``` nano is editor name, you can use vim or some other you feel comfortable with

Add two lines:

```
203.0.113.10 example-hostname.example.com example-hostname #this one is your ip4
2600:3c01::a123:b456:c789:d012 example-hostname.example.com example-hostname #this is your ip6 (can be find in linodes/networking ip6)
```

Add DNS records to the dashboard , click on Domains, your domain and add ```A/AAAA record```. Two times for ip4 and ip6. Hostname is your domain name: example.com


#### Set your timezone (Ubuntu):

```dpkg-reconfigure tzdata``` pick your country, city. To check that is all set type: ```date```

#### Securing your server https://www.linode.com/docs/security/securing-your-server/

- add user (ubuntu):

```adduser example_user```

- add user to sudo group:

```adduser example_user sudo```

Also you can add all permission to the new user by typiing: ```visudo```
Under `User privilage specification`, add your new user name to have all permissions as root:

to look like this:

```
# User privilege specification
root    ALL=(ALL:ALL) ALL
anita   ALL=(ALL:ALL) ALL
```

Also disable root to noe be able to login by editing ```nano /etc/ssh/sshd_config```  ```PermitRootLogin no```

To change user from root:

```
root@API_ADDRESS: su - anita
```

This is bare minimum of security, for production app you need to follow all recommended steps, like 2 Factor authentication etc.. 

Nevertheless we have all set up to start deployment process. 


#### Install apache server https://www.linode.com/docs/web-servers/apache/apache-web-server-on-ubuntu-14-04/#before-you-begin

Install apache:

- update your system: ```sudo apt-get update && sudo apt-get upgrade```

- install apache: ```sudo apt-get install apache2 apache2-doc apache2-utils```

#### Configure your Linode for Deployment

- install git, yarn, nodejs , you can install nvm to handle node versions

```sudo apt install git```
```sudo apt-get install -y nodejs```
```sudo apt install build-essential```
- yarn
```curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -```
```echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list```
``` sudo apt update && sudo apt install yarn```
``` yarn --version```


#### Create host directory

This is where you app will be clone to:

```mydomain.com``` is your domain name

```sudo mkdir -p /var/www/mydomain.com```

Add permission to that folder to your user: 

```sudo chown -R $USERNAME:$USERNAME /var/www/mydomain.com```
```sudo chmod -R 755 /var/www/mydomain.com```


Clone your app to /var/www/mydomain.com:

sudo git clone `yoururl .` - . means it will unpack all files directly in mydomain.com
run ```yarn install```

Configure your apache server:

```sudo touch /etc/apache2/sites-available/mydomain.com.conf```

Create link from sites-enabled to sites-avalailable

``` sudo ln -s /etc/apache2/sites-available/mydomain.com.conf /etc/apache2/sites-enabled/mydomain.com.conf```

Install few apache mods: ```sudo a2enmod proxy proxy_http rewrite headers expires```

Disable default Apache virtual host ```sudo a2dissite 000-default.conf```

Open ```nano /etc/apache2/sites-available/mydomain.com.conf```

```
<VirtualHost *:80>
  ServerAdmin youremail@example.com
  ServerName  example.com
  ServerAlias www.example.com
 
  DocumentRoot /var/www/example.com
  
  ProxyRequests Off
   ProxyPreserveHost On
   ProxyVia Full
   <Proxy *>
    Require all granted
   </Proxy>

   <Location />
      ProxyPass "http://127.0.0.1:3000/"
      ProxyPassReverse "http://127.0.0.1:3000/"
   </Location>

</VirtualHost>
```

Enable the site: ```sudo a2ensite example.com.conf```

Restart apache: ```sudo systemctl restart apache2```

If you have some issue with restarting the server, you can check whats wrong by typing: ```sudo systemctl status apache2```

#### Set up firewall

```
sudo apt-get install ufw -y
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw status
sudo ufw --force enable
```

#### Run your app

Navigate to ```/var/www/example.com```

You should have all your project there and run what ever script you have to start project, currently only dev script, to run prod script you would need to enable ssl/https ceritficates first.

In my case I have in package.json:

```"scripts": {
    "dev": "NODE_ENV=development npm-run-all --parallel dev:*",
    "dev:server": "nodemon --watch build --exec \"node build/bundle.js\"",
    "dev:build-server": "webpack --config webpack.server.js --watch",
    "dev:build-client": "webpack --config webpack.client.js --watch",
    "prod:build": "npm run build && NODE_ENV=production forever start index.js",
    "staging:build": "npm run build && NODE_ENV=staging node index",
    "build": "cross-env NODE_ENV=production webpack --config webpack.production.js --progress"
  },
  ```
  
  ```yarn dev```, go to browesel and type your api address http://111.111.111:3000 - my port is 3000
  

#### Point your domain to Linode

You need to do 2 steps, one in GoDaddy (I have domain in godaddy, if you bought domian somewhere else then you need to configure it there) and second step is in Linode dashboard.

GoDaddy instructions:
Go to your domain and click on Manage DNS
Then click on ```Enter my own nameservers (advanced):``` and add linode servers:

```
ns1.linode.com
ns2.linode.com
ns3.linode.com
ns4.linode.com
ns5.linode.com
```

Then go to Linode and create A record for your domain:

Navigate to your domains in Cloud Manager --> https://cloud.linode.com/domains
Click on the domain you want to add the A record for
Click 'Add an A/AAAA Record'
Leave the Hostname blank, add your Linode's IP, then click 'Save'

It can take up to few hours to enable your domain.

#### Next step SSL certificate


