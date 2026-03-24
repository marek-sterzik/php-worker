# the envcheck command

Inside of the container, there is one special command `envcheck` defined. `envcheck` is used to check and also modify environment variables. While checking environment variables is supported from any script, modifying environment variables is possible only in bash scripts provided by the fact that `source envcheck` statement was invoked before the envcheck call. (`source envcheck` is automatically run inside the script calling `bin/docker-entrypoint` and therefore this statement does not need to be here)

## basic usage

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

## check definitions

### any

Don't perform any real check. Just declare that this variable may conatin any value.

```bash
envchekck NOTE=any
```

### array

Check if the environment vairable is a space-separated array. A subcheck may be defined for each element of the array as an argument.

```bash
envcheck URLS=array:url:schemes=https
```

### bool

Check if value is a boolean. `true`, `false`, `yes`, `no`, `1`, `0` are understood as boolean values. The variable is modified to only values `1` or `0`.

```bash
envcheck DEBUG_ENABLED=bool
```

### credentials

Check if the variable is a pair of an username and password splitted by a comma. Username and password are understood to be uri-encoded and may therefore contain a comma as well.
This checker, if successful, is creating two more variables with a suffix `_USERNAME` and `_PASSWORD` containing the already uri-decoded username and password.

```bash
envcheck ACCESS_CREDENTIALS=credentials

# if this check is successful, it will also create a variable `ACCESS_CREDENTIALS_USERNAME` and `ACCESS_CREDENTIALS_PASSWORD`
```

### default

Set the variable value to a default value if the variable is not defined or empty.

```bash
envcheck APP_NAME=default:"Cool App"
```

### delete

Unset the variable

```bash
envcheck SENSITIVE_VARIABLE=delete
```

### domain

Check if the variable is a vaild domain

```bash
envcheck SERVER_DOMAIN=domain
```

### enum

Check if the variable is one of the variables given as arguments

```bash
envcheck MODE=enum:master:slave
# allows values "master" and "slave" as the content of the MODE variable
```

### exists

Check if the variable is defined (even empty variable may be defined, therefore this definition differs from `nonempty`).

```bash
envcheck NOTE_REQUIRED=exists
# we want fail if NOTE_REQUIRED variable is missing, but allow to explicitely define the variable as an empty string
```

### file

Check, if the variable represents a valid filename.

```bash
envcheck CONFIG_FILE=file
```

### hostname

Check, if the variable is a valid hostname. IP address is not allowed as a valid hostname.

```bash
envcheck MY_HOST=hostname
```

### host

Check, if the variable is a valid hostname or IP address.

```bash
envcheck DB_HOST=host
```

### hostport

Check, if the variable is a valid pair `host:port`. For example `myworker:80`.

```bash
envcheck REMOTE_SERVICE=hostport
```

### identifier

Check, if the variable is a valid c-like identifier.

```bash
envcheck REMOTE_SERVICE=identifier
```

### int

Check if the variable is a valid integer.

```bash
envcheck NUM_ATTEMPTS=int
envcheck NUM_ATTEMPTS=int:1:5 # NUM_ATTEMPTS must be between 1 and 5
```

### ip

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

### ipv4

Alias for check `ip:v4`.

```bash
envcheck IPV4=ipv4
```

### ipv6

Alias for check `ip:v6`.

```bash
envcheck IPV6=ipv6
```

### lc

Convert variable to lowercase.

```bash
envcheck MODE=lc
```

### ne

Check that value is not equal to the given value.

```bash
envcheck HOST=ne:localhost
```

### nonempty

Check if the value of the variable is non-empty (differs from check `exists`).

```bash
envcheck SECRET=nonempty
```

### port

Check if the value is a valid TCP or UDP port number.

```bash
envcheck PORT=port
```

### re

Check the variable according to a PCRE regexp.

```bash
envcheck ID=re:'^[a-z]+$'
```

### uc

Convert variable to uppercase.

```bash
envcheck MODE=uc
```

### uint

Check if the variable is an unsigned integer. Range may be specified in the same way as in `int`.

```bash
envcheck ID_RECORD=uint
envcheck NUMBER_OF_ATTEMPTS=uint:2:10
```

### url

Check if the variable is a valid URL. Many options are available, but will be described later.

```bash
envcheck REMOTE_URL=url
```

