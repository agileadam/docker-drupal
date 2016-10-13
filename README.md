Drupal development with Docker
==============================

_This is a modified README from the forked repo_

Quick and easy to use Docker container for your *local Drupal development*. It contains a LAMP stack and an SSH server, along with an up to date version of Drush. It is based on [Debian Jessie](https://wiki.debian.org/DebianJessie).

Summary
-------

This image contains:

* Apache 2.4
* MySQL 5.5
* PHP 5.6
* Drush 7
* Drush 8
* Drupal 7
* Composer
* PHPMyAdmin
* Blackfire

When launching, the container will contain a fully-installed, ready to use Drupal site.

### Passwords

* Drupal: `suser:admin`
* MySQL: `root:` (no password)
* SSH: `root:root`

### Exposed ports

* 80 and 443 (Apache)
* 22 (SSH)
* 3306 (MySQL)

### Environment variables

If you wish to enable [Blackfire](https://blackfire.io) for profiling, set the following environment variables:

* `BLACKFIREIO_SERVER_ID`: Your Blackfire server ID
* `BLACKFIREIO_SERVER_TOKEN`: Your Blackfire server token

Tutorial
--------

You can read more about the original image [here](http://wadmiraal.net/lore/2015/03/27/use-docker-to-kickstart-your-drupal-development/).

Installation
------------

### Github

Clone the repository locally and build it:

	git clone https://github.com/agileadam/docker-drupal.git
	cd docker-drupal
	git checkout 7.x
	docker build -t agileadam/drupal-7.51 .

Notice that there are several branches. The `master` branch always refers to the current recommended major Drupal version (version 8 at the time of writing). Other branches, like `7.x`, reflect prior versions. You can also build from a particular tag by checking out that tag before using `docker build` (e.g., `git checkout 7.41`).


Running it
----------

The container exposes its `80` and `443` ports (Apache), its `3306` port (MySQL) and its `22` port (SSH). Make good use of this by forwarding your local ports. You should at least forward to port `80` (using `-p local_port:80`, like `-p 8080:80`). A good idea is to also forward port `22`, so you can use Drush from your local machine using aliases, and directly execute commands inside the container, without attaching to it.

Here's an example just running the container and forwarding `localhost:8080` and `localhost:8022` to the container:

	docker run -d --name mycontainer -p 8080:80 -p 8022:22 -t agileadam/drupal-7.51

If you want to run in HTTPS, you can use:

	docker run -d --name mycontainer -p 8443:443 -p 8022:22 -t agileadam/drupal-7.51

### Pre-installed Drupal versus Using existing files/db

The containers you create using this image will have Drupal installed and ready-to-go (see credentials above).

*If you want to just use the environment with your own existing code/db*:

Map the volume to the Drupal site directory on your host machine

	docker run -d --name mycontainer -p 8080:80 -p 8022:22 -v /Users/adam/phpstorm/mysite:/var/www -t agileadam/drupal-7.51

Change `settings.php` as needed to use the `root` mysql credentials:

	vim /Users/adam/phpstorm/mysite/sites/default/settings.php

Import your database

	TBD


### Writing code locally

If you map the entire Drupal site as a volume (shown above and on the next line) you will already be able to write code locally.

	docker run -d --name mycontainer -p 8080:80 -p 8022:22 -v /Users/adam/phpstorm/mysite:/var/www -t agileadam/drupal-7.51

Here's another example. This time we're running the container, forwarding port `8080` like before, but also mounting Drupal's `sites/all/modules/custom/` folder to the local `modules/` folder. You can then start writing code on your local machine, directly in this folder, and it will be available inside the container:

	docker run -d --name mycontainer -p 8080:80 -p 8022:22 -v `pwd`/modules:/var/www/sites/all/modules/custom -t agileadam/drupal-7.51

### Using Drush

Using Drush aliases, you can directly execute Drush commands locally and have them be executed inside the container.

*This docker image uses Drush 7 by default, but drush7 and drush8 binaries both exist on the image so you can use 7 or 8 to connect.*

Create a new aliases file in your home directory and add the following:

	# ~/.drush/docker.aliases.drushrc.php

	<?php
	if (!isset($drush_major_version)) {
		$drush_version_components = explode('.', DRUSH_VERSION);
		$drush_major_version = $drush_version_components[0];
	}

	$aliases['mycontainer'] = array(
	  'root' => '/var/www',
	  'remote-user' => 'root',
	  'remote-host' => 'localhost',
	  'ssh-options' => '-p 8022', // the port you specified when running the container
	  'path-aliases' => array(
	    '%drush-script' => 'drush' . $drush_major_version,
	  )
	);

Next, if you do not wish to type the root password everytime you run a Drush command, copy the content of your local SSH public key (usually `~/.ssh/id_rsa.pub`; read [here](https://help.github.com/articles/generating-ssh-keys/) on how to generate one if you don't have it). SSH into the running container:

	# If you forwarded another port than 8022, change accordingly.
	# Password is "root".
	ssh root@localhost -p 8022

Once you're logged in, add the contents of your `id_rsa.pub` file to `/root/.ssh/authorized_keys`. Exit.

You should now be able to call `drush sa` to view available aliases, then you can use the alias like this:

	drush @docker.mycontainer cc all

This will clear the cache of your Drupal site. All other commands will function as well.

NOTE: `drush sql-sync` will not work if both systems are "remote". You can try putting the "source" alias on the docker container by ssh'ing into it and putting it there, but YMMV. A simple solution (though probably not idea for large databases) is to pipe the sql-dump from the source into the destination like this (source on left, destination on right):

	drush @somesource sql-dump | drush @docker.mycontainer sql-cli

### Running tests

If you want to run tests, you may need to take some additional steps. Drupal's Simpletest will use cURL to simulate user interactions with a freshly installed site when running tests. This "virtual" site resides under `http://localhost:[forwarded ip]`. This gives issues, though, as the *container* uses port `80`. By default, the container's virtual host will actually listen to *any* port, but you still need to tell Apache on which ports it should bind. By default, it will bind on `80` *and* `8080`, so if you use the above examples, you can start running your tests straight away. But, if you choose to forward to a different port, you must add it to Apache's configuration and restart Apache. You can simply do the following:

	# If you forwarded to another port than 8022, change accordingly.
	# Password is "root".
	ssh root@localhost -p 8022
	# Change the port number accordingly. This example is if you forward to port 8081.
	echo "Listen 8081" >> /etc/apache2/ports.conf
	/etc/init.d/apache2 restart

Or, shorthand:

	ssh root@localhost -p 8022 -C 'echo "Listen 8081" >> /etc/apache2/ports.conf && /etc/init.d/apache2 restart'

If you want to run tests from HTTPS, though, you will need to edit the VHost file `/etc/apache2/sites-available/default-ssl.conf` as well, and add your port to the list.

### MySQL and PHPMyAdmin

PHPMyAdmin is available at `/phpmyadmin`. The MySQL port `3306` is exposed. The root account for MySQL is `root` (no password).

### Blackfire

[Blackfire](https://blackfire.io) is a free PHP profiling tool. It offers very detailed and comprehensive insight into your code. To use Blackfire, you must first register on the site. Once registered, you will get a *server ID* and a *server token*. You pass these to the container, and it will fire up Blackfire automatically.

Example:

	docker run -it --rm -e BLACKFIREIO_SERVER_ID="[your id here]" -e BLACKFIREIO_SERVER_TOKEN="[your token here]" -p 8022:22 -p 8080:80 agileadam/drupal-7.51

You can now start profiling your application.
