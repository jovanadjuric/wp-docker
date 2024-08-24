# WordPress Docker development template

This template is made for local development only.

WordPress Core files are moved into a separate folder in this setup.

For the WordPress core, install path is `src/wp` folder, but the wp-config file used is in the root (
customized with `bin/setup-wp` script, see later).

To put WP core to separate folder set `WP_CORE_SEPARATE` env variable to `"true"` (**DO NOT change it**)

_Note: Previously, there was the option to set up WordPress the usual way (no separate wp folder), but it is removed
now._

The following Docker images are included:

- [wordpress:php8.2-apache](https://hub.docker.com/_/wordpress)
- [nginx:latest](https://hub.docker.com/_/nginx)
- [mysql:8.0](https://hub.docker.com/_/mysql)
- [phpmyadmin/phpmyadmin](https://hub.docker.com/r/phpmyadmin/phpmyadmin)
- [sj26/mailcatcher:v0.10.0](https://hub.docker.com/r/sj26/mailcatcher)


Change your docker PHP version in: `.docker/images/wordpress/Dockerfile`, or `.docker/images/wordpress/Dockerfile-ssl`.
It works with 8.0, 8.1 as well. WordPress 6.6 [already supports PHP 8.2]((https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/)) (with possible exceptions)

**Project structure:**

- `.docker`: contains all docker-specific files and folders
- `src`: the source code goes here


## Install a new, or setup an existing WordPress site

### HTTPS install

Use the `master` branch which is the default!

1. Set your environment variables in `.env`, change `APP_NAME` (also read the 5th point below!)
2. Setup ssl for your custom domain `${APP_NAME}.local`

```shell
bin/setup-ssl
```

You may need to make scripts in the `bin` folder, and the `.docker/images/nginx/ssl/mkcert-v1.4.3-linux-amd64`
executable (https://github.com/FiloSottile/mkcert/releases).

The supplied mkcert version is for Debian/Ubuntu.

3. Modify `.docker/images/nginx/conf.d/default.conf`: Change `server_name` to your custom domain (`${APP_NAME}.local`)!

4. Add domain alias for 127.0.0.1 (e.g. `nano /etc/hosts` for Linux)

5. Build docker project (on Linux make sure to configure Docker so that you can run it without sudo privileges):

```shell
(set -a;source .env;docker-compose -f docker-compose-ssl.yml up --build)
```

With sudo privileges (not recommended):

```bash
set -a
source .env
sudo -E docker-compose up --build
```

5. WordPress database installation with wp-cli, import db for existing sites

- For a new site run this command (it will set up your database and configure your site automatically):

```shell
bin/setup-wp
```

- For an existing site, import sql dump or use PhpMyAdmin (however, the latter is not suitable for importing large
  database sql dumps):

```shell
bin/mysql-import
```

The sql dump should be placed into this folder with the same filename: `db/db.sql`

Make sure to have the correct database name in the comments of the sql dump. Example:

```sql
-- MySQL dump 10.13  Distrib 8.0.31, for Linux (x86_64)
--
-- Host: localhost    Database: testdbname
--
-- Current Database: `testdbname`
--
CREATE
DATABASE /*!32312 IF NOT EXISTS*/ `testdbname` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci */ /*!80016 DEFAULT ENCRYPTION='N' */;
USE
`testdbname`;
```

For existing sites, replace domain in database tables (enter container first with bash):

```shell
bin/bash
wp search-replace 'https://example.com' 'https://example.local' --skip-columns=guid
```

**Add your themes, plugins and other assets to `src\wp-content/`**

No need to change any additional files. The `wp-config.php` loads all credentials from the `.env` file. Every bash
script in bin folder loads environment variables from `.env`.

Create a new admin user if needed, example:

```shell
bin/bash
wp user create username name@example.com --role=administrator --user_pass=Password123!
```


### HTTP install

Installation is almost the same as for https. We just need to use another docker-compose file for the build:

```shell
(set -a;source .env;docker-compose -f docker-compose.yml up --build)
```


## Configure WordPress local site

1. Install/update composer packages (composer.json is in the `/src` folder). Run

```shell
bin/composer update
```

_! NOTICE: Installing or updating WordPress with composer returns a non-breaking error. However, plugins and themes are
installed properly despite the error. Need to be resolved. -> error DISAPPEARED (2023-01-25)_

2. Add/update `wp-config.php` constants, values etc. (optional, not needed generally)

```bash
# List all currently defined configuration props 
wp config list

# Update salts used for hashing
wp config shuffle-salts

# Show debug information
wp config set WP_DEBUG true --raw
wp config set WP_DEBUG_LOG true --raw
wp config set WP_DEBUG_DISPLAY true --raw

# Force 'direct' filesystem method for WP (automatic detection doesn't work well in a Docker container)
wp config set FS_METHOD "'direct'" --type=constant --add --raw

# Disable core/plugin/theme modification
wp config set "DISALLOW_FILE_MODS" true --type=constant --add --raw
```


## Customisations made to `wordpress` image

- wp-cli, and composer 2 was installed. (The official image does not have it. There is a `wordpress:cli` image, but it
  only contains the wp-cli. In this image, apache2 is also configured. This is the reason why it is used here.)
- For convenient work in the terminal, vim and nano is also installed.
- `php.ini` setting change for mailcatcher (you can change the email to any fake one)
  `sendmail_path = /usr/bin/env catchmail -f wordpress@local.test`
- upload_max_filesize = 20M


## Composer

Core, plugins and themes are installed/handled/updated/deleted with composer to keep track of the dependency versions.

Do not update packages with the wp-cli, because it will lead to inconsistencies in versions defined in
composer.json and the actual versions installed!

Modifying files on the wp-admin can be disabled (using the `DISABLE_FILE_MODS` constant) that will make changes to core,
themes and plugins impossible on the
admin dashboard:

```bash
bin/bash
wp config set "DISALLOW_FILE_MODS" true --type=constant --add --raw
```

Update packages:

```bash
bin/composer update
```


## MySQL database management, PhpMyAdmin

- Import database: `bin/mysql-import`
- Export database: `bin/mysql-dump`

All sql files should be put into the `db` folder.

Note: Auto-importing db dump is currently disabled. In `docker-compose.yml`, this is commented out:
`./db:/docker-entrypoint-initdb.d # where to find the db dump data`

It is a better idea to use the `bin/mysql-import` script instead.

Make sure to replace urls in data tables (like https://example.com -> https://example.local). The easiest way is to use
the wp-cli:

```shell
bin/bash
wp search-replace 'https://example.com' 'https://example.local' --skip-columns=guid
```

Change 'wp_' table prefix to a custom one. For example, use PhpMyAdmin: select all data tables and change table prefix
on the
UI. [Read more](https://help.one.com/hc/en-us/articles/360002107438-Change-the-table-prefix-for-WordPress-).
In addition, you need to update the meta_keys in these tables (there may be more keys you need to change):

```sql
UPDATE NEWPREFIX_usermeta
SET meta_key = 'NEWPREFIX_capabilities'
WHERE meta_key = 'OLDPREFIX_capabilities';
UPDATE NEWPREFIX_usermeta
SET meta_key = 'NEWPREFIX_user_level'
WHERE meta_key = 'OLDPREFIX_user_level';
UPDATE NEWPREFIX_usermeta
SET meta_key = 'NEWPREFIX_autosave_draft_ids'
WHERE meta_key = 'OLDPREFIX_autosave_draft_ids';
UPDATE NEWPREFIX_options
SET option_name = 'NEWPREFIX_user_roles'
WHERE option_name = 'OLDPREFIX_user_roles';
```

PhpMyAdmin service is accessible here: `http://localhost:1337`


## Manage docker containers / Development

Make sure the host ports that are bound to our container's internal ports are not in use by another container or host
process.

The containers need to be built for the first time:

```bash
(set -a;source .env;docker-compose -f docker-compose.yml up --build)
```

```bash
(set -a;source .env;docker-compose -f docker-compose-ssl.yml up --build)
```

- Start containers: `bin/start`
- Stop containers: `bin/stop`
- Down containers (this will remove and destroy containers): `bin/down`
- Restart containers: `bin/restart`

```bash
docker-compose down
```


## Bash scripts in `bin` folder

Useful and makes it faster to run some common tasks. Check them out.


## Useful WP CLI commands

[Full documentation.](https://developer.wordpress.org/cli/commands/)

```shell
# CORE
wp help
wp core is-installed
wp core version
wp core check-update
#wp core update -> use composer instead
#wp core download -> use composer instead
wp core update-db


# REPLACE domain name - very useful
# Search and replace but skip one column
wp search-replace 'http://example.test' 'https://example.com' --skip-columns=guid

# Create a POT file for the WordPress plugin/theme into the specified directory
wp i18n make-pot plugins/your-plugin-name plugins/your-plugin-name/languages/your-textdomain.pot

# Call get_bloginfo() to get the name of the site.
wp shell
get_bloginfo( 'name' );

# Flushes the object cache.
wp cache flush

# Flush rewrite rules
wp rewrite flush

# Update permalink structure
wp rewrite structure '/%year%/%monthnum%/%postname%'

# List rewrite rules
wp rewrite list --format=csv


# CONFIG
# Shuffle the salts for the hashes:
wp config shuffle-salts
wp config list

# show debug information
wp config set WP_DEBUG true --raw
wp config set WP_DEBUG_LOG true --raw
wp config set WP_DEBUG_DISPLAY true --raw

# force 'direct' filesystem method for WP (automatic detection doesn't work well in a Docker container)
wp config set FS_METHOD "'direct'" --type=constant --add --raw

# disable core/plugin/theme modification
wp config set "DISALLOW_FILE_MODS" true --type=constant --add --raw


# PLUGINS
# Activate plugin
wp plugin activate hello

# Deactivate plugin
wp plugin deactivate hello

# Delete plugin
# wp plugin delete hello -> use composer instead

# Install the latest version from wordpress.org and activate
# wp plugin install bbpress --activate  -> use composer instead

# Generate a new plugin with unit tests
wp scaffold plugin sample-plugin


# THEMES
# Generate theme based on _s
wp scaffold _s sample-theme --theme_name="Sample Theme" --author="John Doe"

# Generate code for post type registration in given theme
wp scaffold post-type movie --label=Movie --theme=simple-life

# Get status of theme
wp theme status twentysixteen
wp theme activate

# Install the latest version of a theme from wordpress.org and activate
# wp theme install twentysixteen --activate  -> use composer instead

# Get details of an installed theme
wp theme get twentysixteen --fields=name,title,version
```


## Deployment on shared hosting

1. Create a new MySQL database
2. Update `wp-config` with the new database credentials (database name, username, password). The host is `"localhost"`
   most of
   the time.
3. Export your local database with PhpMyAdmin, or create a sql dump with MySQL in the terminal (for larger databases,
   the latter is recommended). After that, import it into your live database.
4. You have to rewrite the urls and some options with your live domain name (see `src/data/rewrite-domainname.sql`). Or,
   you
   can also use the `wp-cli` locally, and then export your local database (and rewrite them back to the local ones
   afterward).
5. Copy the content of the `src/` folder to your host through SFTP.
6. Post-deployment: Setup SMTP, increase security (use WordFence firewall), etc.

Occasionally, change your [salts](https://api.wordpress.org/secret-key/1.1/salt/) used for hashing.


## Update .gitignore file!

Make sure you NEVER commit the .env file and wp-config.php to version control. It also applies to your ssl
certificate keys you generated with mkcert!

These ignore rules are already added to `.gitignore`.

**Good tip: Set up GitGuardian** to get notification about potential credential leaks.


## Mailcatcher for local e-mail testing

1. See the `mailcatcher` in the docker-compose files.

2. Install `"wpackagist-plugin/wp-mail-smtp":"4.0.1"` with Composer inside the WordPress service if you don't have it:
   `bin/composer require "wpackagist-plugin/wp-mail-smtp":"4.0.1"`.

3. Configure SMTP:

Setup "from address", and "name". Choose the "Other SMTP" option.
Set these:

- SMTP Host: **mailcatcher**
- SMTP Port: **1025**

No need for encryption.
Auto TLS -> Off.
Authentication -> None.

4. Send out a test email (Tools menu point under WP Mail SMTP).

5. Access Mailcatcher UI at `localhost:1080`

Generally, there is no need to configure anything else if you followed the instructions, or you didn't make other
changes. See the PHP section on the [mailcatcher website](https://mailcatcher.me/), or in
the [PHP documentation](https://www.php.net/manual/en/mail.configuration.php#ini.sendmail-path) for more info.

This is the `php.ini` file location inside the **wordpress** container: `/usr/local/etc/php`. This file is copied here
from `.docker/images/wordpress/php.ini-development`.

This is the default setting for `sendmail_path`:
`sendmail_path = /usr/bin/env catchmail -f wordpress@local.test`

### Possible issues with catchmail

**/usr/bin/env: 'catchmail': No such file or directory**

You can try to change the setting to this ([source](https://wesamly.wordpress.com/2015/10/27/mailcatcher-error-env-catchmail-no-such-file-or-directory/)):
`sendmail_path = /usr/bin/env /usr/local/bin/catchmail -f some@example.com`

Or you can use mhsendmail as an alternative with this setting:
`sendmail_path = /usr/local/bin/mhsendmail`

[msendmail install instructions.](https://github.com/SalsaBoy990/docker-wordpress/issues/4#issuecomment-2016944122)


## License, credits

MIT License
Copyright (c) 2021-2023 András Gulácsi
