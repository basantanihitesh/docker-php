[![Build Status](https://travis-ci.org/bryanlatten/docker-php.svg?branch=master)](https://travis-ci.org/bryanlatten/docker-php)
[![Docker Pulls](https://img.shields.io/docker/pulls/bryanlatten/docker-php.svg?maxAge=2592000)]()

docker-php
==========

Provides a pre-wired, configurable PHP + Nginx setup across multiple runtime versions.

Integrated with Behance’s [docker-nginx](https://github.com/behance/docker-nginx)

Available on [Docker Hub](https://hub.docker.com/r/bryanlatten/docker-php/). 

### Quick-start

- `docker run bryanlatten/docker-php:7.0 "php" "-v"`
- `docker run bryanlatten/docker-php:7.1 "php" "-v"`
- `docker run bryanlatten/docker-php:7.1-alpine "php" "-v"`
- `docker run bryanlatten/docker-php:7.2-beta "php" "-v"`

Adding code to runtime, see [here](https://github.com/bryanlatten/docker-php#expectations).  
PHP tuning and configuration, see [here](https://github.com/bryanlatten/docker-php#downstream-configuration).  
Nginx tuning and configuration, see [here](https://github.com/behance/docker-nginx#environment-variables).  
Adding startup logic, [basic](https://github.com/behance/docker-base#startupruntime-modification) or [advanced](https://github.com/behance/docker-base#advanced-modification).  

#### Container tag scheme: `PHP_MAJOR.PHP_MINOR(-Major.Minor.Patch)(-variant)`

- `PHP_MAJOR.PHP_MINOR`, required. Engine versions of PHP. ex. `docker-php:7.1`
- `(Major.Minor.Patch)`, optional. Semantically versioned container provisioning code. ex. `docker-php:7.1-12.4.0`.  
- `(-variant)`, optional. Alpine variants are slim versions of the container. ex. `docker-php:7.1-alpine`.  

### Includes
---

- [Nginx](https://github.com/behance/docker-nginx) HTTP server
- PHP / PHP-FPM: primary runtime
- [S6](https://github.com/just-containers/s6-overlay): PID 1 zombie reaping, startup coordination, shutdown signal transferal. Nginx and PHP are preconfigured to shutdown as gracefully as possible.
- [Goss](https://goss.rocks): for serverspec-like testing. Run `goss -g /tests/php-fpm/{PHP_MAJOR.PHP_MINOR}(-variant).goss.yaml` to validate any configuration updates
- Ubuntu (default) or Alpine OS [base](https://github.com/behance/docker-base)
- Extra PHP Modules:

`*`  - not available on `-alpine` variant  
`^`  - not available on `7.2` 
`~`  - disabled by default (default: use `phpenmod` to enable, Alpine-only: uncomment .ini file)

  - apcu
  - bcmath
  - bz2
  - calendar
  - ctype
  - curl
  - date
  - dom
  - exif
  - cgi-fcgi
  - gd
  - gearman*
  - iconv
  - igbinary
  - intl
  - json
  - mbstring
  - mcrypt^
  - memcache*
  - memcached
  - msgpack
  - mysqli
  - mysqlnd
  - newrelic~^ (activates with env variables)
  - opcache
  - openssl
  - pcntl
  - pdo
  - pdo_mysql
  - pdo_pgsql~
  - pgsql~
  - phar
  - posix
  - redis~
  - shmop
  - SimpleXML
  - sockets
  - sysvmsg
  - sysvsem
  - sysvshm
  - tokenizer
  - xdebug~^
  - xml
  - xmlreader
  - xmlwriter
  - yaml~
  - zip
  - zlib



### Expectations
---

Sample `Dockerfile`
```
FROM bryanlatten/docker-php:7.1

# (optional, recommended) Verify everything is in order from the parent
RUN goss -g /tests/php-fpm/7.1.goss.yaml validate && /aufs_hack.sh

# Layer local code into runtime
COPY ./ /app/

# Done!
```

- Local code should be copied into `/app`, for example:
```COPY ./ /app/```
- Nginx is pre-configured to use a front controller PHP file  (entrypoint) 
a front controller called `index.php` within a `public` folder. `/app/public/index.php`

- Dev Mode (no ENV variables): PHP's opcache is enabled, and is set to check files for updates. Code can be developed locally in Docker by mounting into the `/app` folder. 
For example, the `docker-compose.yml` syntax: 
```  
volumes:
   - ./:/app
```
- Production Mode [recommended]: using ENV variable, `CFG_APP_DEBUG=0`. Container becomes immutable, PHP's opcache is configured to not check files for updates.
- NOTE: Nginx is exposed and bound to an unprivileged port, `8080`.  

### Monitoring
--- 
- NewRelic APM: automatically enabled by adding providing environment variables `REPLACE_NEWRELIC_APP` and `REPLACE_NEWRELIC_LICENSE`
- PHP-FPM Status: available *only* inside container at `/__status`. Application healthcheck can pull PHP-FPM statistics from `http://127.0.0.1/__status?json`. To open to more clients than local, add more `allow` statements in `__status` location block in `$CONF_NGINX_SITE`(`/etc/nginx/sites-available/default`)
- Nginx Status: available *only* inside container at `/__nginx_status`. Application healthcheck can pull nginx statistics from `http://127.0.0.1/__nginx_status`. To open to more clients than local, add more `allow` statements in `__nginx_status` location block in $CONF_NGINX_SITE (`/etc/nginx/sites-available/default`) 

### Downstream Configuration
---
Several environment variables can be used to configure various PHP FPM paramaters, as well as a few Nginx configurations.
as such. These can be used to drive the configuration of the downstream PHP application in any way necessary, but there are a few environment variables that `bryanlatter/docker-php` will process along the way...

See parent(s) [docker-nginx](https://github.com/behance/docker-nginx), [docker-base](https://github.com/behance/docker-base) for additional configuration


Variable | Example | Default | Description
--- | --- | --- | ---
`*` | `DATABASE_HOST=master.rds.aws.com` | - | PHP has access to environment variables by default
`CFG_APP_DEBUG` | `CFG_APP_DEBUG=1` | 1 | Setting to `1` or `true` will cue the Opcache to watch for file changes. Set to 0 for *production mode*, which provides a sizeable performance boost, though manually updating a file will not be seen unless the opcache is reset.
`CFG_XDEBUG_ENABLE` | `CFG_XDEBUG_ENABLE=1` | - | Setting to `1` or `true` will enable the XDebug extension, which is preconfigured to allow remote debugging as well as profiling. NOTE: Requires "dev" mode be enabled via `CFG_APP_DEBUG`.
`SERVER_MAX_BODY_SIZE` | `SERVER_MAX_BODY_SIZE=4M` | 1M | Allows the downstream application to specify a non-default `client_max_body_size` configuration for the `server`-level directive in `/etc/nginx/sites-available/default`
`SERVER_FASTCGI_BUFFERS` | `SERVER_FASTCGI_BUFFERS=‘512 32k’` | 256 16k | [docs](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_buffers), [tweaking](https://gist.github.com/magnetikonline/11312172#determine-actual-fastcgi-response-sizes)
`SERVER_FASTCGI_BUFFER_SIZE` | `SERVER_FASTCGI_BUFFER_SIZE=‘256k’` | 128k | [docs](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_buffers_size), [tweaking](https://gist.github.com/magnetikonline/11312172#determine-actual-fastcgi-response-sizes)
`SERVER_FASTCGI_BUSY_BUFFERS_SIZE` | `SERVER_FASTCGI_BUSY_BUFFERS_SIZE=‘1024k’` | 256k | [docs](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_busy_buffers_size)
`REPLACE_NEWRELIC_APP` | `REPLACE_NEWRELIC_APP=prod-server-abc` | - | Sets application name for newrelic
`REPLACE_NEWRELIC_LICENSE` | `REPLACE_NEWRELIC_LICENSE=abcdefg` | - | Sets license for newrelic, when combined with above, will enable newrelic reporting
`PHP_FPM_MEMORY_LIMIT` | `PHP_FPM_MEMORY_LIMIT=256M` | 192MB | Sets memory limit for FPM instances of PHP
`PHP_FPM_MAX_EXECUTION_TIME` | `PHP_FPM_MAX_EXECUTION_TIME=30` | 60 | Sets time limit for FPM workers
`PHP_FPM_UPLOAD_MAX_FILESIZE` | `PHP_FPM_UPLOAD_MAX_FILESIZE=100M` | 1M | Sets both upload_max_filesize and post_max_size
`PHP_FPM_MAX_CHILDREN` | `PHP_FPM_MAX_CHILDREN=15` | 4096 | [docs](http://php.net/manual/en/install.fpm.configuration.php)
`PHP_FPM_START_SERVERS` | `PHP_FPM_START_SERVERS=40` | 20 | [docs](http://php.net/manual/en/install.fpm.configuration.php)
`PHP_FPM_MAX_REQUESTS` | `PHP_FPM_MAX_REQUESTS=100` | 1024 | [docs](http://php.net/manual/en/install.fpm.configuration.php) How many requests an individual FPM worker will process before recycling
`PHP_FPM_MIN_SPARE_SERVERS` | `PHP_FPM_MIN_SPARE_SERVERS=10` | 5 | [docs](http://php.net/manual/en/install.fpm.configuration.php)
`PHP_OPCACHE_MEMORY_CONSUMPTION` | `PHP_OPCACHE_MEMORY_CONSUMPTION=512` | 128 | [docs](http://php.net/manual/en/opcache.configuration.php#ini.opcache.memory-consumption)
`PHP_OPCACHE_MAX_WASTED_PERCENTAGE` | `PHP_OPCACHE_MAX_WASTED_PERCENTAGE=10` | 5 | [docs](http://php.net/manual/en/opcache.configuration.php#ini.opcache.max-wasted-percentage)
`PHP_OPCACHE_INTERNED_STRINGS_BUFFER` | `PHP_OPCACHE_INTERNED_STRINGS_BUFFER=64` | 16 | [docs](http://php.net/manual/en/opcache.configuration.php#ini.opcache.interned-strings-buffer)

### Testing
---   
- Requires `docker` and `docker-compose`   
To test locally, run `bash -e ./test.sh {docker-machine}` where `docker-machine` is the IP of the connected docker engine. 
These same tests get run automatically, per pull request, via Travis CI

