---
layout: post
title: "Setup Node&Mongo API on Ubuntu"
date: 2018-08-22 18:00:00 +0200
categories: blog
description: Steps on how to setup test and production environments for a Node.js api with MongoDB on Ubuntu
keywords: Ubuntu, MongoDB, Node, Express, Git, pm2, Nginx, setup, development, production
---

In this post I will go through the steps on how to configure a very simple setup with test and production environments for a Node api on an Ubuntu server. 

The main idea is to run a single server that exposes two instances of the api - one for test and one for production purposes. Then the consuming application can use the test version of the api under development and the production one in production environment.

What we are going to do here in a nutshell: install `MongoDB` and create test and production databases, install `Node.js`, fetch the api on the server machine, install process manager `pm2` and run it on the api, install and configure `nginx` as a reverse proxy for `pm2`. In the end there will be test and production versions of the api sitting on the same domain but different url paths and they will be automatically started in case of expected/unexpected server reboot.

This setup isn't meant for a large "production"-like project but it worked really well for my small hobby project which is a `Node.js` API in `GraphQL` and `Express` and a `MongoDB` database. So if you have a small project with a similar stack of technologies and looking for a quick and simple setup you might want to stick around here for a while :)

Before we begin configuring the server we need to have the api switch connection strings to the db depending on the environment and be able to run the api on different ports. This is a typical use-case for environmental variables.

## Configure environmental variables in the api

Environmental variables in `Node` are accessed via `process.env` object. So if we want to pass port as an env variable we will use it like this:

```javascript
const PORT = process.env.PORT || 4000;
app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`);
});
```

Now we need a way to have different connection strings to the `mongo` db depending on the value of `NODE_ENV` which will be set to `development` or `production`. We will set those connection strings as project's specific environmental variables in `.env` file and load them into `process.env` using an `npm` package [`dotenv`](https://github.com/motdotla/dotenv). So first install the package:
```
npm install --save dotenv
```
and create `.env` in the root of the project with two connection strings, for example:

```
MONGO_URL_DEV="mongodb://localhost:27017/my-db-development"
MONGO_URL_PROD="mongodb://localhost:27017/my-db-production"
```
Your `.env` might contain very sensitive information (like username/password to the db) so it is **extremely** important to add it to `.gitignore`. Just add a line with `.env` to your `.gitignore` and you are safe :)

According to the [dotenv docs](https://github.com/motdotla/dotenv#usage), this line should appear as early as possible in the code:

```javascript
require('dotenv').config();
```

so we will have it as first line in the file that initializes the node server (usually `index.js`, `server.js` or `app.js`).

To read the connection strings we can now access `process.env.MONGO_URL_DEV` and `process.env.MONGO_URL_PROD`. We want to use the correct url depending on the value of `NODE_ENV` and to encapsulate this logic we can create a `config` file or add it to the existing `config`:

```javascript
module.exports = {
  development: {
    url: process.env.MONGO_URL_DEV
  },
  production: {
    url: process.env.MONGO_URL_PROD
  }
};
```

And fetch the right connection string where we establish connection to the database:

```javascript
const nodeEnv = process.env.NODE_ENV || 'development';
const { MongoClient } = require('mongodb');

// select the configuration for the current node environment
const mongoConfig = require('../config/mongo')[nodeEnv];

MongoClient.connect(mongoConfig.url, (err, mPool) => {
  // express code: define routes, headers etc.
  ...
});
```
If you want to check how this is implemented in my api, check the full code on my [github](https://github.com/margaretkrutikova/words-api-graphql).

The rest of the time we are going to spend in the terminal one-by-one with the ubuntu server :)

## Connect to the server

You should already have a server with a clean `Ubuntu` installed and open ports for `HTTP` and `SSH` connections. The actual steps in this post were performed on an `Azure` virtual machine running `Ubuntu 18.04` (in case of creating a new virtual machine on `Azure` there is a step allowing to select public inbound ports of which `HTTP` and `SSH` should be selected). In the terminal write:

```bash
ssh username@ip-address
```
*enter correct password*

Phew.. the toughest part is now behind. Get the system updates:
```bash
sudo apt-get update
```

## Install and start MongoDB

There is an official guide on how to install the `MongoDB` packages on `Ubuntu` available on [mongodb docs](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/), some of the steps have specific commands depending on the version of `Ubuntu`.

After you have completed all the steps, start mongo with
```bash
sudo service mongod start`
```
and make sure it is running by issuing:
```bash
sudo service mongod status
```

To make `mongo` wake up on system startup run:
```bash
sudo systemctl enable mongod.service
```

## Create test and prod databases

Let's connect to `mongo` shell and check what databases it has:
```bash
mongo
show dbs
```

Now create a test db and collections:
```bash
use my-db-development
db.createCollection("things")
```

Should return `{"ok": 1}`

Create a production db and collections:
```bash
use my-db-production
db.createCollection("things")
```
Once again should say `{"ok": 1}`

Type `exit` and let's go on to the next step.

## Install and configure Node.js

If you try simply running `sudo apt install nodejs`, it will leave you with some really old version of `node`, so this path is not recommended. At the time of writing the latest stable version of `node` is 8.11.4 (includes npm 5.6.0) and it can be installed from a `NodeSource` binary distribution.

Go to home directory with `cd ~` and get the binaries (the steps borrowed from [this nodejs documentation](https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions)):

```bash
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt-get install -y nodejs
```
Verify that running `nodejs -v` gives the latest version of `node.js`.

## Setup the api

Install `Git`:

```bash
sudo apt-get install git
```
and confirm it with `git --version`.

Navigate to your user's home directory and create a folder for your api:

```bash
cd ~
mkdir my-things-api
cd my-things-api
```

Clone the repository into two locations `development` and `production`.
```bash
git clone https://github.com/your-username/my-things-api.git development
git clone https://github.com/your-username/my-things-api.git production
```

Now, under `development` you might want to `git checkout` your `dev` branch and under `production` your `master` branch. This depends on the git workflow you have chosen for your project. 

Run `npm install` in each folder and create a `.env` file with `touch .env`. Open the file with your favorite text editor (`vim .env`) and paste the connection strings we created previously into the file, for example:

```
MONGO_URL_DEV="mongodb://localhost:27017/my-db-development"
MONGO_URL_PROD="mongodb://localhost:27017/my-db-production"
```

Now we can start and stop the `express` server just to see that it doesn't crash:
```bash
cd ~/my-things-api
NODE_ENV=production node production/path_to_the_server/app.js
NODE_ENV=development node development/path_to_the_server/app.js
```

## Install process manager pm2

Install `pm2` and set it to wake up on server restart:
```bash
sudo npm install -g pm2
pm2 startup
```

Run the command generated by `startup` to finish startup configuration.

Let's create a configuration file for `pm2` with two app instances which have `NODE_ENV` set to `development` and `production`. Generate a default file in the home directory with the api and edit it to set our configuration:

```bash
cd ~/my-things-api
pm2 ecosystem
vim ecosystem.config.js
```

Here is the configuration I used for the setup:

```
module.exports = {
  apps : [{
    name      : 'my-things-api',
    script    : './development/path_to_the_server/app.js',
    watch: true,
    env: {
      NODE_ENV: 'development'
    }
  },{
    name      : 'my-things-api',
    script    : './production/path_to_the_server/app.js',
    watch: true,
    env: {
      NODE_ENV: 'production',
      PORT: 4010
    }
  }]
};
```

It is extremely simplistic and you might want to add paths to `error_file` and `out_file` to log errors and output, more attributes can be found in the [pm2 docs](http://pm2.keymetrics.io/docs/usage/application-declaration/#attributes-available).

Start `pm2` and check the list of processes it started:
```bash
pm2 start ecosystem.config.js
pm2 list
```

Two instances of the api should now be running. To check run:
```bash
curl http://localhost:4000
curl http://localhost:4010
```

Finally save the started applications so that they can be restarted on server restart
```bash
pm2 save
```

At this point I started thinking about setting up `Jenkins` to poll the repository for changes (or using github hooks) to automatically deploy new versions of the development and production branches, but that felt like too much for my small project. You are free to investigate this topic and tell about the consequences :)

## Install and configure nginx

We will use `nginx` as a reverse proxy that will accept requests from the network and pass them down to the internal api running on `nodejs` and managed by `pm2`.  

A good guide on how to install `nginx` is available on [digital ocean](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04). Or follow those steps:
```bash
sudo apt install nginx
sudo ufw allow 'Nginx HTTP'
sudo ufw status
sudo systemctl status nginx
```

Now let's enable nginx on system startup
```bash
sudo systemctl enable nginx
```

To configure nginx to act a reverse proxy we need to add some configuration. For simplicity we will edit the default `nginx` configuration file:
```bash
sudo vim /etc/nginx/sites-available/default
```

Let's assume you want to access your api from `/things-api/development` and `/things-api/production`. In this case add the following configuration to the file after the first `location` section:

```
location /things-api/development/ {
        proxy_pass http://localhost:4000/;   # note the trailing slash!
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
}
location /things-api/production/ {
        proxy_pass http://localhost:4010/;  # note the trailing slash!
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
}
```

It is extremely important to have the trailing slash in `proxy_pass` since this will make `nginx` pass the rest of the url path after the api path in your node application. 

You might also want to disable the default `nginx` start page, which can be done by editing the section with `location /`:

```
location / {
  return 404;
}
```

All the rest in the config file is intact. Save and close the file. 

Check whether syntax in the config file is correct and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Time to log out, run `exit`.

## Final test

The api should now be running under `http://server-domain-name/things-api/development/` and `http://server-domain-name/things-api/production/` and you should be able to test it in the browser or `postman`.

If you are brave enough, restart the server and check that everything is going to start up correctly after the reboot :)


