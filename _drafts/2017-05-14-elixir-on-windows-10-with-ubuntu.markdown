---
layout: post
title:  "Installing Elixir on Windows 10 Creators Update with Ubuntu on WSL"
date:   2017-05-14 16:24:00
categories: wsl
---

I've been following Josh Adam's Elixir episodes on DailyDrip as he builds the Firestorm forum. For the exercises, I decided to setup my Elixir dev environment on Windows 10, but using Windows Subsystem for Linux, which is working much better after Creators Update.

The following worked for me, but no idea if these are best practices:

1. Install Erlang and Elixir

{% highlight bash %}
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install elixir

2. Setup Mix

{% highlight bash %}
mix local.hex

3. Install Phoenix

{% highlight bash %}
mix archive.install https://github.com/phoenixframework/archives/raw/master/phoenix_new.ez

4. Install other dependencies: intofy-tools, postgresql, nodejs

{% highlight bash %}
sudo apt-get install inotify-tools
sudo apt-get install postgresql
sudo apt-get install nodejs

5. Install and configure npm

{% highlight bash %}
sudo apt-get install npm
sudo ln -s /usr/bin/nodejs /usr/bin/node
npm install

PostgreSQL wouldn't play ball for some reason, so I changed the auth method for user (= "postgres") by editing (= "/etc/postgresql/9.5/main/pg_hba.conf")...

{% highlight ApacheConf %}
\# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                md5

... and setting up its password:

{% highlight bash %}
psql -U postgres
\password
\q

# Creating and configuring a new Phoenix project

1. Create the project

{% highlight bash %}
mix phx.new my_project
cd my_project

2. Create the database

{% highlight bash %}
mix ecto.create

PostgreSQL complained of unmatching template encodings, so I edited (= "my_project/config/dev.exs") to force ( ="template0"). Alternatively, I could set it as default in PostgreSQL.

{% highlight Elixir %}
# Configure your database
config :my_project, MyProject.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "my_project_dev",
  hostname: "localhost",
  template: "template0",
  pool_size: 10

2. Resolve dependencies

{% highlight bash %}
cd assets/
npm install