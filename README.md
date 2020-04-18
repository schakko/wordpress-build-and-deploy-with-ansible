# What is this all about?
Keeping your local WordPress development in sync with the remote state sucks. Plug-ins like WP Migrate DB and others do only work for databases but don't fit into any build process. From my perspective, the best strategy is using Docker to make the WordPress environment immutable.
Nevertheless, there are a few drawbacks, even with Docker:

- New plug-ins must be (de-)activated in the production environment
- Themes must be (re-)activated

Besides that, for smally deployments we are using [Uberspace](https://uberspace.de/) as hosting provider for WordPress and not AWS or GCP. Uberspace doesn't provide any Docker support (at the moment). Because of this, we are using Ansible to deploy the local development enviroment to the production environment.

# Important
- On Ubuntu / WSL you have to use python-mysqldb and *not* python-pymsql. The later one expects a valid socket which is not available if MySQL or MariaDB are running in Windows.

# Setup
## Local project directory setup
This repository will only work for you, if your project directory and WordPress instance has a setup like mine. We are distinguishing between WordPress (*src/*), the build/deploy environment (*this* repository) and environment specfic configuration (*config/*).

Besides that, our WordPress setup itself looks differently:

1. We are using a different WordPress directory layout to distinguish between the WordPress core itself and installed plug-ins. The WordPress core goes to *src/wordpress*. Just unzip the WordPress ZIP file in there.
2. [DotEnv](https://symfony.com/doc/current/components/dotenv.html) is used for loading the WordPress configuration and be independent from the underlying host.
3. Composer is used to install any dependencies like DotEnv. Ansible autodetects if *composer.json* is present inside the *src* directory and runs the install process

These are the steps to setup the WordPress environment:


	$ mkdir my-uberspace
	$ mkdir -p my-uberspace/{src/vendor,src/wp-content,src/wordpress,config/environment/prod,config/credentials}
	# download WordPress ZIP distribution and unzip it to src/wordpress
	$ cp src/wordpress/wp-config-sample.php src/wp-config.php
	# checkout this project
	$ git clone https://github.com/schakko/wordpress-build-and-deploy-with-ansible.git my-uberspace/build-and-deploy
	# copy my-uberspace/build-and-deploy/environment/sample config/environment/prod
	$ cp build-and-deploy/environment/sample config/environment/prod
	# Ansible will stop, if the file .env does not exist inside the target's src/ directory.
	# 1. If you want to use DotEnv, the file with all configuration parameter has to be created after the first deployment
	# 2. If you don't want to use DotEnv, just create an empty file:
	$ touch src/.env

You'll end up with the following directory structure:

	/build-and-deploy			# This Git repository
	/src					# WordPress source goes here, managed by own git repository
	/src/vendor				# Composer dependencies, if used
	/src/wordpress				# WordPress core
	/src/wp-content				# WordPress externals, e.g. plug-ins or themes
	/src/.env				# DotEnv configuration file
	/config/environment/prod		# Ansible production environment goes here, managed by own git repository
	/config/credentials			# Credentials, e.g. PEMs go here, should not be managed by git


## Install wp-cli


    $ curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
    $ chmod +x wp-cli.phar

    # local
    $ mv wp-cli.phar /usr/local/bin/wp
    
    # on Uberspace
    $ mv wp-cli.phar ~/bin/wp

    
# Usage
Note: `$ANSIBLE` is a shortcut for `ansible-playbook -i ../config/environment/prod`.

## How do i ...

| Task | Command | Note |
| --- | --- | --- |
|create a remote database dump and download it?|`$ANSIBLE pull.yml --tags make-database-snapshot`|Snapshot can be found in the *.download* directory|
|update local WordPress and all plug-ins?|`$ANSIBLE -i ../config/environment/$ENV -l local update.yml`||
|deploy my local WordPress instance o remote?|`$ANSIBLE -i ../config/environment/$ENV deploy.yml`||

## Details
### Fetch - Sync remote (PROD) database and files to local system
It
- downloads the latest MySQL backup
- imports the latest MySQL dump to the local database; *DROP TABLE IF EXISTS* is enabled

```
	$ $ANSIBLE -i ../config/environment/$ENV fetch.yml
	# if you also want to include uploaded files (images etc.), use the *with-uploads* tag
	$ $ANSIBLE -i ../config/environment/$ENV --tags with-uploads fetch.yml
```

### Build only
The *build.yml* create the .htaccess file for the local environment, activates the theme and all plug-ins defined in the `local.yml` file


	$ $ANSIBLE build.yml
	
### Deploy - Builds remote environment and synchronizes local state to remote state
It
- creates the .htaccess file of the remote instance
- copies the local WordPress files (themes, plug-ins) to the remote instance
- enables the defined theme and plug-ins in the remote instance

```
	$ $ANSIBLE -i ../config/environment/$ENV deploy.yml
```

### Pull only - pulls the remote data
The *pull.yml* only retrieves the remote files, without intefering with WordPress itlsef


	$ $ANSIBLE -i ../config/environment/$ENV pull.yml
	
	
## Local deployment
To deploy the .htaccess file to the local installation directory, you need to specify the endpoint with *-l local*:

    $ $ANSIBLE -i ../config/environment/$ENV -l local deploy.yml
