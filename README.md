# php-worker

This is an universal docker image to run php applications. The image does not work as a standalone container, but expect a php application to be mounted inside of the container.

The worker listens on standard http port 80. Https protocol is not directly supported or you will need to use some reverse proxy before this container.

The http server is based on nginx. Php uses the php-fpm fastcgi module to execute php scripts. 

It is configured in this way:

* All files in the document root (see later) and its subdirectories are always served as static files.
* Non-existent files are routed to the default php script (index.php by default) also located in the document root.
* All other logic may be build up on this base.

## interface between php-worker and the application

* The whole application code is expected to be inside of the `/app` dir, which is empty by default.
* The default document root is the subdirectory `public/` of `/app`, but this may be changed using environment variables (see later)
* In the document root there should be located a php script being the entrypoint of all requests not corresponding to statically served files. This scirpt is `index.php` by default and can be also changed using environment variables.
* In the `bin/` directory there may be located 3 special files:
  - `docker-entrypoint` which **must** be a bash script and is responsible for setting up all environment variables. This scirpt is always invoked with root privileges.
  - `docker-boot-root` - which is run after all environment variables are already set, but before the webserver itself is started. It may contain any code initializing the application. The script may be any unix executable including scripts (must have permissions to execute) and is invoked as root.
  - `docker-boot` - same as `docker-boot-root`, but the script is invoked with the permissions of the user of the webserver (which may be also configured using environment variables). If both - `docker-boot-root` and also `docker-boot` are defined, both are executed. `docker-boot-root` first and then `docker-boot`.
* The container itself expects, that all persistent data will be stored in the `/persistent` directory. But setting up any docker volume on this dir is the responsibility of the user using this container.

### setting environment variables

Environment variables should be processed in the `bin/docker-entrypoint` script which must be a bash script and is sourced from the main entrypoint script. This `docker-entrypoint` script:

* has available all environment variables passed to the docker container (except environment variables explicitely used by the php-worker itself)
* may do any chanes to these variables, which are then available through the whole php-worker code
* may set special environment variables which are changing the behavior of the php-worker itself
* may use the `envcheck` command to proceed sophisticated checks on the environment variables (for example envcheck may check if an environment variable does contain an url and it may also parse this url) The `envcheck` command will be described later.
* all the environment variables set by this script will be available in the `docker-boot-root` and `docker-boot` scripts and also when executing custom commands (see later)

These environment variables are processed by the worker itself and are fine-tuning the behavior of the worker:

* `WORKER_ROOT` - Defines the php worker document root relatively to the application dir. Default `public`.
* `WORKER_PHP_SCRIPT` - Defines the php script acting as an php entrypoint invoked in case of a dynamically processed http requests. The value is understood relatively to the worker's document root and only scripts being directly in the document root are allowed. PHP entrypoints being located in subdirectories of the document root were never tested.
* `SERVER_UID` - The uid or username of the user running the webserver. The user itself does not need to exist in the container. It will work also with uids not defined in `/etc/passwd`. Default `www-data`.
* `SERVER_GID` - The gid or groupname of the group running the webserver. The group itself does not need to exist in the container. It will work also with gids not defined in `/etc/group`. Default `www-data`.
* `XDEBUG_PORT` - The port where xdebug should connect to. Ip addresss is now hardcoded to the docker host Ip. If `XDEBUG_PORT` is set, XDEBUG is enabled. If it is not set, XDEBUG is disabled. Default not set.
* `CRON_JOBS` - If non-empty, it may contain a configuration for standard unix cron. In this case, cron will be invoked and running the jobs defined in this configuration. If empty, cron will not be started at all. Default empty.
* `MAX_BODY_SIZE` - The maximal size of the http body. Valid values are numbers (in bytes) or numbers with the `k` suffix (in kilobyes) or numbers with the `m` suffix (in megabytes). There are also two special values: `default` (default max body size) and `unlimited` - no limit to the body size is set.
* `MEMORY_LIMIT` - Sets the PHP memory limit. Same rules holds as for `MAX_BODY_SIZE` with the same special values `default` and `unlimited`.
* `ACCESS_LOG_FILE` - Path where the nginx access log file is located. Default `/var/log/nginx/access.log`.
* `ERROR_LOG_FILE` - Path where the nginx error log file is located. Default `/var/log/nginx/error.log`.
* `FPM_ACCESS_LOG_FILE` - Path where the php-fpm access log file is located. Default `/var/log/phpfpm/www.access.log`.
* `FPM_ERROR_LOG_FILE` - Path where the php-fpm error log file is located. Default `/var/log/phpfpm/www.error.log`.
* `FPM_SLOW_LOG_FILE` - Path where the php-fpm slow log file is located. Default `/var/log/phpfpm/www.slow.log`.
* `NGINX_RSYSLOG_URL` - URL where nginx is sending access log using rsyslog (does not affect the local log file). Default not set.
* `NGINX_RSYSLOG_URL_ERR` - URL where nginx is sending error log using rsyslog (does not affect the local log file). Default not set.
* `PERSISTENT_DIR` - Sets up the relative directory name inside of `/persistent` where all data will be located. This relative directory will be created, all permissions inside of this directory will be set to `$SERVER_UID:$SERVER_GID` and the variable will be converted to absolute path. For example, if you will set the `PERSISTENT_DIR` to `worker`, the container will create a directory `/persistent/worker` if it does not exist, change all file permissions to `www-data:www-data` (or other uid/gid) inside of this directory and set the variable itself to `PERSISTENT_DIR="/persistent/worker"`.

These variables are processed as bash arrays:

* `SERVER_GIDS` - Use secondary groups or group ids for the user running the web server. Default empty array.
* `WRITABLE_FILES` - sets the names of files (relatively to `/app` directory) which should be made writable by the web server. Directories are made writable recursively. If the directory or file does not exist, they will be created. If the filename in this array ends with `/`, a directory will be created, an empty file otherwise. 
* `WORKER_COMMANDS` - a list of CLI commands in the form `<global_name>:<relative_local_script_name>`. Where `<global_name>` is the global command name being created and `<relative_local_script_name>` is the application's binary which should be invoked. The relative binary is always invoked as using the `SERVER_UID/SERVER_GID` user permissions, even if the `<global_name>` command is invoked as root. For example if you define `console:bin/console`, then in the container a global binary in some of the PATH-listed directory `console` is created. This `console` command will invoke the `/app/bin/console` binary, but will ensure it is run always as the proper user running the webserver. Before invoking `bin/console`, all environment variables set by `bin/docker-entrypoint` are also loaded.

All these variables cannot be set directly using the docker container's environment variables. If you want to control them using docker container's variables, you need to pass the information using some another general-purpose variable and set these control variables using the general-purpose variables.

## other features of the container

* there is always a command `x` executing any command as the webserver user/group and loading `bin/docker-entrypoint` environment variables first. If you run `x` without any argument, it will invoke just bash. This command is useful when executing anything using `docker exec` to ensure a stable environment set by `bin/docker-entrypoint`. (`docker exec` guarantees only environment variables passed to the container itself, but not any futher modifications of these variables). In fact, if you define any command in the array `WORKER_COMMANDS`, it will internally use this `x` command.
* You may always use the hostname `dashost` referring to the docker host IP. It is working according to expectations only on platforms, where docker runs natively (Linux) and may not work as expected on platforms, where docker is run inside some virtual host.

## the envcheck command

Inside of the container, there is one special command `envcheck` defined. `envcheck` is used to check and also modify environment variables. While checking environment variables is supported from any script, modifying environment variables is possible only in bash scripts provided by the fact that `source envcheck` statement was invoked before the envcheck call. (`source envcheck` is automatically run inside the script calling `bin/docker-entrypoint` and therefore this statement does not need to be here)

### basic usage

```bash
source envcheck # ensure we can use envcheck to modify environment variables

envcheck VARIABLE=def:arg1=value1:arg2=value2...
```

This call will check environment variable `$VARIABLE` according to the definition `def` with arguments `arg1=value1`, `arg2=value2`, etc. In fact each definition has just a single argument, but this single argument is usually parsed using a standardized way allowing to use named arguments. But some definitions use another approach.

One `envcheck` call may proceed multiple checks. Each check is a standalone argument of the command. All checks passed to the command must pass to envcheck finish successfully. When any check will fail, the whole envcheck command will fail. When the command fails, it will never modify any variables, even if the particular check invoking a modification was successful.

When you want to proceed two checks on the same variable in the same command, you may replace the variable name by `.` in second and next occurences.

Examples:

```bash
envcheck APP_NAME=default:"Cool App" # set the variable APP_NAME to value "Cool App" if the variable was not yet set.

envcheck APP_SECRET=nonempty # APP_SECRET must be non-empty

envcheck DEBUG_ENABLED=default:true .=bool # first: set value to "true" if DEBUG_ENABLED variable is empty, second: check (and convert) the value of DEBUG_ENABLED as boolean
```

### check definitions

#### any

Don't perform any real check. Just declare that this variable may conatin any value.

```bash
envchekck NOTE=any
```

#### array

Check if the environment vairable is a space-separated array. A subcheck may be defined for each element of the array as an argument.

```bash
envcheck URLS=array:url:schemes=https
```

#### bool

Check if value is a boolean. `true`, `false`, `yes`, `no`, `1`, `0` are understood as boolean values. The variable is modified to only values `1` or `0`.

```bash
envcheck DEBUG_ENABLED=bool
```

#### credentials

Check if the variable is a pair of an username and password splitted by a comma. Username and password are understood to be uri-encoded and may therefore contain a comma as well.
This checker, if successful, is creating two more variables with a suffix `_USERNAME` and `_PASSWORD` containing the already uri-decoded username and password.

```bash
envcheck ACCESS_CREDENTIALS=credentials

# if this check is successful, it will also create a variable `ACCESS_CREDENTIALS_USERNAME` and `ACCESS_CREDENTIALS_PASSWORD`
```

#### default

Set the variable value to a default value if the variable is not defined or empty.

```bash
envcheck APP_NAME=default:"Cool App"
```

#### delete

Unset the variable

```bash
envcheck SENSITIVE_VARIABLE=delete
```

#### domain

Check if the variable is a vaild domain

```bash
envcheck SERVER_DOMAIN=domain
```

#### enum

Check if the variable is one of the variables given as arguments

```bash
envcheck MODE=enum:master:slave
# allows values "master" and "slave" as the content of the MODE variable
```

#### exists

Check if the variable is defined (even empty variable may be defined, therefore this definition differs from `nonempty`).

```bash
envcheck NOTE_REQUIRED=exists
# we want fail if NOTE_REQUIRED variable is missing, but allow to explicitely define the variable as an empty string
```

#### file

Check, if the variable represents a valid filename.

```bash
envcheck CONFIG_FILE=file
```

#### hostname

Check, if the variable is a valid hostname. IP address is not allowed as a valid hostname.

```bash
envcheck MY_HOST=hostname
```

#### host

Check, if the variable is a valid hostname or IP address.

```bash
envcheck DB_HOST=host
```

#### hostport

Check, if the variable is a valid pair `host:port`. For example `myworker:80`.

```bash
envcheck REMOTE_SERVICE=hostport
```

#### identifier

Check, if the variable is a valid c-like identifier.

```bash
envcheck REMOTE_SERVICE=identifier
```

#### int

Check if the variable is a valid integer.

```bash
envcheck NUM_ATTEMPTS=int
envcheck NUM_ATTEMPTS=int:1:5 # NUM_ATTEMPTS must be between 1 and 5
```

#### ip

Check if the variable is a valid IP address.

Check flags may be specified, the order is important:
* `v4` - check if it is a valid ipv4 address
* `v6` - check if it is a valid ipv6 address
* `host4` - check if it is a hostname with an ipv4 record, which will be converted to ipv4
* `host6` - check if it is a hostname with an ipv6 record, which will be converted to ipv6
* `host` - equivalent to `host4:host6` (perform both checks, first the host4, then host6 test)

Other flags (order is not important here):
* `multi` - allow multiple IPs (ips are space separated)
* `range` - allow also IP ranges
* `strict` - strict IP ranges
* `all` - allow a special range `all` or `*` reprezenting the range 0.0.0.0/0

```bash
envcheck IP=ip
envcheck IPV4=ip:v4
envcheck IPV6=ip:v6
envcheck IPV4=ip:v4:host4 # convert hostnames to ipv4 address

envcheck IP_RANGES=ip:multi:range:all
```

#### ipv4

Alias for check `ip:v4`.

```bash
envcheck IPV4=ipv4
```

#### ipv6

Alias for check `ip:v6`.

```bash
envcheck IPV6=ipv6
```

#### lc

Convert variable to lowercase.

```bash
envcheck MODE=lc
```

#### ne

Check that value is not equal to the given value.

```bash
envcheck HOST=ne:localhost
```

#### nonempty

Check if the value of the variable is non-empty (differs from check `exists`).

```bash
envcheck SECRET=nonempty
```

#### port

Check if the value is a valid TCP or UDP port number.

```bash
envcheck PORT=port
```

#### re

Check the variable according to a PCRE regexp.

```bash
envcheck ID=re:'^[a-z]+$'
```

#### uc

Convert variable to uppercase.

```bash
envcheck MODE=uc
```

#### uint

Check if the variable is an unsigned integer. Range may be specified in the same way as in `int`.

```bash
envcheck ID_RECORD=uint
envcheck NUMBER_OF_ATTEMPTS=uint:2:10
```

#### url

Check if the variable is a valid URL. Many options are available, but will be described later.

```bash
envcheck REMOTE_URL=url
```
