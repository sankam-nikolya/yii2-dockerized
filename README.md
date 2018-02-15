Yii 2 Dockerized
================

A template for docker based Yii 2 applications.

 * Ephemeral container, configured via environment variables
 * Application specific base image
 * Optional local configuration overrides for development/debugging (git-ignored)
 * Base scaffold code for login, signup and forgot-password actions
 * Flat configuration file structure

# 1 Main Concepts

## 1.1 Base Image

The core idea of this template is that you build a bespoke **base image**
for your application that meets your project's exact requirements. This image
contains:

 * PHP runtime environment (e.g. Apache + PHP module)
 * PHP extensions
 * Composer packages

The base image will hardly ever change unless

 * you want to upgrade to a newer PHP or Yii version or
 * you want to add a missing PHP extension or composer package.

Its configuration can be found in the `./build` directory:

 * `Dockerfile` adds PHP extensions and required system packages
 * `composer.json` and `composer.lock` list composer packages

The actual **app image** extends from this base image and uses `./Dockerfile`
in the main directory. It basically only adds your app sources on top.

In the recommended scenario you would build the base image once then upload
it to your container registry and share it with your co-developers.

If you don't have a container registry you can still use our template. But then
each developer in your team will have to build the same base image locally.

## 1.2 Runtime Configuration via Environment Variables

We follow docker's principle that containers are configured via environment
variables. This only includes runtime configuration, of course: Things like
whether to run in debug mode or DB credentials. Most other settings like for
example URL rules will be hardcoded in `./config/web.php`.

You should continue to follow this principle when developing your app. For
more details also see our
[yii2-configloader](https://github.com/codemix/yii2-configloader) that we use
in this template.


# 2 Initial Project Setup

## 2.1 Download Application Template

First fetch a copy of our application template, for example with git:

```sh
git clone https://github.com/codemix/yii2-dockerized.git myproject
rm -rf myproject/.git
```

You could also download the files as ZIP archive from GitHub.

## 2.2 Update/Add Composer Packages

We use a composer container that is based on the official
[composer](https://hub.docker.com/r/library/composer/) image.

If your app needs some additional composer packages besides yii2 or
if you want to update composer packages, go to the `./build` directory
of the app:

```sh
cd myproject/build
```

To add a package run:

```sh
sudo docker-compose run --rm composer require some/library
```

To update all packages run:

```sh
sudo docker-compose run --rm composer update
```

This will update `composer.json` and `composer.lock` respectively. You can
also run other composer commands, of course.

**You have to rebuild your base image afterwards** (see below).


> **Note:** As docker's composer image may not meet the PHP requirements of all
> your packages you may have to add `--ignore-platform-reqs` to be able to
> install some packages.


## 2.3 Build the Base Image

Before you continue with building the base image you should:

 * Set a tag name for the base image in `./build/docker-compose.yml`
 * Use the same tag name in `./Dockerfile` in the main directory
 * Optionally add more PHP extensions or system packages in `./build/Dockerfile`

To build the base image, again go to the `./build` directory and run:

```sh
sudo docker-compose build
```

Now you could upload that image to your container registry.

## 2.4 Cleanup and Initial Commit

At this point you may want to modify our application template, add some default
configuration and remove those parts that you don't need. Afterwards you are
ready for the initial commit to your project repository.


# 3 Local Development

During development we map the local app directory into the app image.
This way we always run the code that we currently work on.

As your local docker setup may differ from production (e.g. use different
docker network settings) we usually keep `docker-compose.yml` out of version
control. Instead we provide a `docker-compose-example.yml` with a reasonable
example setup.

Since the runtime configuration should happen with environment variables, we
use a `.env` file. We also keep this out of version control and only include a
`.env-example` to get developers started.

## 3.1 Initial Setup of a Development Environment

After you've cloned your project you need to prepare two files:

```sh
cd myproject
cp docker-compose-example.yml docker-compose.yml
cp .env-example .env
```

You should modify these two files e.g. to enable debug mode or configure a
database. Then you should be ready to start your container and initialize
your database (if your project has one):

```sh
sudo docker-compose up -d
# Wait some seconds to let the DB container fire up ...
sudo docker-compose exec web ./yii migrate
```

When done, you can access the new app from
[http://localhost:8080](http://localhost:8080).

> **Note:** If you see an error about write permissions to `web/assets/`,
> `runtime/` or `var/sessions` it's because the local file owner id is
> different from `1000` which is the `www-data` user inside the container.
>
> To fix this, run:
>
>     sudo docker-compose exec web chown www-data web/assets runtime var/sessions
>

## 3.2 Development Session

A development session will usually go like this:

```sh
sudo docker-compose up -d
# edit files, check, tests, ...
git add .
git commit -m 'Implemented stuff'
sudo docker-compose stop
```

> **Note:** Another approach is to leave a terminal window open and start the
> container with
>
>     sudo docker-compose up
>
> This way you can always follow the live logs of your container. To stop all
> containers, press `Ctrl-c`.

### 3.2.1 Running yiic Commands

If you need to run yiic commands you execute them in the running development
container:

```sh
sudo docker-compose exec web ./yiic migrate/create add_column_name
```

> **Note:** If a command creates new files on your host (e.g. a new migration)
> you may have to change file permissions to make them writeable on your host
> system:
>
>     chown -R mike migrations/*
>

### 3.2.2 Adding or Updating Composer Packages

The procedure for adding or updating composer packages is the same as described
in 2.2 above. Remember that you have to rebuild the base image afterwards. It
should probably also receive an updated version tag.

### 3.2.3 Working with Complex Local Configuration

Sometimes you may have to add complex parts to `config/web.php` but want
to avoid the risk of accidentally committing them. You therefore can activate
support for a `config/local.php` file in your `.env`. This config file will
be merged into `config/web.php`.

```sh
# Whether to load config/(local|console-local).php. Default is 0.
ENABLE_LOCALCONF=1
```

### 3.2.3 Make Composer Packages Available Locally

For some IDEs it's useful to have the composer packages available on your local
host system. To do so you can simply copy them from inside the container:

```
sudo docker-compose exec web cp -rf /var/www/vendor ./
```

> **Note:** Inside the container composer packages live in `/var/www/vendor`
> instead of the app's `./vendor` directory. This way we don't override the
> vendor directory when we map the local app directory into the container.
