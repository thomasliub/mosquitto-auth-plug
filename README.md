# mosquitto-auth-plug( A project to add topic wildchar support for redis auth backend for [mosquitto-auth-plug](https://github.com/jpmens/mosquitto-auth-plug))

This project is forked from [mosquitto-auth-plug](https://github.com/jpmens/mosquitto-auth-plug) to expand the topic wildchar support for redis auth backend.

This is a plugin to authenticate and authorize [Mosquitto] users from one
of several distinct back-ends:

* MySQL
* PostgreSQL
* CDB
* SQLite3 database
* [Redis] key/value store
* TLS PSK (the `psk` back-end is a bit of a shim which piggy-backs onto the other database back-ends)
* LDAP
* HTTP (custom HTTP API)
* JWT
* MongoDB
* Files

## Introduction

This plugin can perform authentication (check username / password)
and authorization (ACL). Currently not all back-ends have the same capabilities
(the the section on the back-end you're interested in).

| Capability                 | mysql | redis | cdb   | sqlite | ldap | psk | postgres | http | jwt | MongoDB | Files |
| -------------------------- | :---: | :---: | :---: | :---:  | :-:  | :-: | :------: | :--: | :-: | :-----: | :----:
| authentication             |   Y   |   Y   |   Y   |   Y    |  Y   |  Y  |    Y     |  Y   |  Y  |  Y      | Y
| superusers                 |   Y   |       |       |        |      |  3  |    Y     |  Y   |  Y  |  Y      | N
| acl checking               |   Y   |   Y   |   2   |   2    |      |  3  |    Y     |  Y   |  Y  |  Y      | Y
| static superusers          |   Y   |   Y   |   Y   |   Y    |      |  3  |    Y     |  Y   |  Y  |  Y      | Y

 1. Topic wildcards (+/#) are not supported
 2. Currently not implemented; back-end returns TRUE
 3. Dependent on the database used by PSK

Multiple back-ends can be configured simultaneously for authentication, and they're attempted in
the order you specify. Once a user has been authenticated, the _same_ back-end is used to
check authorization (ACLs). Superusers are checked for in all back-ends.
The configuration option is called `auth_opt_backends` and it takes a
comma-separated list of back-end names which are checked in exactly that order.

```
auth_opt_backends cdb,sqlite,mysql,redis,postgres,http,jwt,mongo
```

Note: anonymous MQTT connections are assigned a username of configured in the
plugin as `auth_opt_anonusername` and they
are handled by a so-called _fallback back-end_ which is the *first* configured
back-end.

Passwords are obtained from the back-end as a PBKDF2 string (see section
on Passwords below). Even if you try and store a clear-text password,
it simply won't work.

The mysql back-end supports expansion of `%c` and `%u` as clientid and username
respectively. This allows ACLs in the database to look like this:

```
+-----------+---------------------------------+----+
| username  | topic                           | rw |
+-----------+---------------------------------+----+
| bridge-01 | $SYS/broker/connection/%c/state |  2 |
+-----------+---------------------------------+----+
```

The plugin supports so-called _superusers_. These are usernames exempt
from ACL checking. In other words, if a user is a _superuser_, that user
doesn't require ACLs.

A static superuser is one configured with the _fnmatch(3)_ `auth_opt_superusers`
option. The other 'superusers' are configured (i.e. enabled) from within the
particular database back-end. Effectively both are identical in that ACL
checking is disabled if a user is a superuser.

Note that not all back-ends currently have 'superuser' queries implemented.
todo. At that point the `auth_opt_superusers` will probably disappear.

## Building the plugin

In order to compile the plugin you'll require a copy of the [Mosquitto] source
code together with the libraries required for the back-end you want to use in
the plugin. OpenSSL is also required.

Copy `config.mk.in` to `config.mk` and modify `config.mk` to suit your building environment, in particular, you have
to configure which back-ends you want to provide as well as the path to the
[Mosquitto] source and its library.

After a `make` you should have a shared object called `auth-plug.so`
which you will reference in your `mosquitto.conf`.

Note that OpenSSL as shipped with OS X is probably too old. You may wish to use a version
supplied by home brew or build your own, and then adapt `OPENSSLDIR` in `config.mk`.

## Configuration

The plugin is configured in [Mosquitto]'s configuration file (typically `mosquitto.conf`),
and it is loaded into Mosquitto auth the ```auth_plugin``` option.


```
auth_plugin /path/to/auth-plug.so
```

Options therein with a leading ```auth_opt_``` are handed to the plugin. The following
"global" ```auth_opt_*``` plugin options exist:

| Option         | default    |  Mandatory  | Meaning               |
| -------------- | ---------- | :---------: | --------------------- |
| backends       |            |     Y       | comma-separated list of back-ends to load |
| superusers     |            |             | fnmatch(3) case-sensitive string
| log_quiet      | false      |             | don't log DEBUG messages |
| cacheseconds   |                   |             | Deprecated. Alias for acl_cacheseconds
| acl_cacheseconds  | 300               |             | number of seconds to cache ACL lookups. 0 disables
| auth_cacheseconds | 0                 |             | number of seconds to cache AUTH lookups. 0 disables
| acl_cachejitter   | 0                 |             | maximum number of seconds to add/remove to ACL lookups cache TTL. 0 disables
| auth_cachejitter  | 0                 |             | maximum number of seconds to add/remove to AUTH lookups cache TTL. 0 disables
=======

Individual back-ends have their options described in the sections below.

There are two caches, one for ACL and another for authentication. By default only the ACL cache is enabled.

After a backend responds (postitively or negatively) for an ACL or AUTH lookup, the result will be kept in cache for
the configured TTL, the same ACL lookup will be served from the cache as long as the TTL is valid.
The configured TTL is the auth/acl_cacheseconds combined with a random value between -auth/acl_cachejitter and +auth/acl_cachejitter.
For example, with an acl_cacheseconds of 300 and acl_cachejitter of 10, ACL lookup TTL are distributed between 290 and 310 seconds.

Set auth/acl_cachejitter to 0 disable any randomization of cache TTL. Settings auth/acl_cacheseconds to 0 disable caching entirely.
Caching is useful when your backend lookup is expensive. Remember that ACL lookup will be performed for each message which is sent/received on a topic.
Jitter is useful to reduce lookup storms that could occur every auth/acl_cacheseconds if lots of clients connect at the same time (for example
after a server restart, all your clients may reconnect immediately and all may cause ACL lookups every acl_cacheseconds).

### Redis
```
auth_opt_redis_userquery GET USER:%s
auth_opt_redis_aclquery GET ACL:%s
```

In `auth_opt_redis_userquery` the parameter is the _username_, whereas in `auth_opt_redis_aclquery`, the parameter is also the _username_.
For ACL query the key is a hash key. The field of the key is the mqtt topic(with wildchar allowed). And the value of the key is the ACL level 1 or 2.

For example forllowing redis auth backend configurations:

Get user password.
```
>>> import redis
>>> r1=redis.Redis()
>>> r1.get('USER:thomas')
'PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8'
```

Get user ACL info.
```
>>> import redis
>>> r1=redis.Redis()
>>> r1.hgetall('ACL:thomas')
{'/#': '1', '/+/device': '2'}
```

If no options are provided then it will default to not using an ACL and using the above userquery.


| Option         | default           |  Mandatory  | Meaning     |
| -------------- | ----------------- | :---------: | ----------  |
| redis_host     | localhost         |             | hostname / IP address
| redis_port     | 6379              |             | TCP port number |

## Configure Mosquitto

```
listener 1883

auth_plugin /path/to/auth-plug.so
auth_opt_redis_host 127.0.0.1
auth_opt_redis_port 6379

# Usernames with this fnmatch(3) (a.k.a glob(3))  pattern are exempt from the
# module's ACL checking
auth_opt_superusers S*
```

## ACL

In addition to ACL checking which is possibly performed by a back-end,
there's a more "static" checking which can be configured in `mosquitto.conf`.

Note that if ACLs are being verified by the plugin, this also applies to
Will topics (_last will and testament_). Failing to correctly set up
an ACL for these, will cause a broker to silently fail with a 'not
authorized' message.

Users can be given "superuser" status (i.e. they may access any topic)
if their username matches the _glob_ specified in `auth_opt_superusers`.

In our example above, any user with a username beginning with a capital `"S"`
is exempt from ACL-checking.

## PUB/SUB

At this point you ought to be able to connect to [Mosquitto].

```
mosquitto_pub  -t '/location/n2' -m hello -u n2 -P secret
```

## PSK

If [Mosquitto] has been built with PSK support, and _auth-plug_ has been built
with `BE_PSK` defined, it supports authenticating PSK connections over TLS, as
long as Mosquitto is appropriately configured.

The way this works is that the `psk` back-end actually uses one of _auth-plug_'s
other databases (`mysql`, `sqlite`, `cdb`, etc.) to obtain the pre-shared key
from the "users" query, and it uses the same database's back-end for performing
authorization (aka ACL checks).

Consider the following `mosquitto.conf` snippet:

```
...
auth_opt_psk_database mysql
...
listener 8885
psk_hint hint1
tls_version tlsv1
use_identity_as_username true
```

TLS PSK is available on port 8885 and is activated with, say,

```
mosquitto_pub -h localhost -p 8885 -t x -m hi --psk-identity ps2 --psk 020202
```

The `use_identity_as_username` option has _auth-plug_ see the name `ps2` as the
username, and this is given to the database back-end (here: `mysql`) to look up
the password as defined for the `mysql` back-end. _auth-plug_ uses its `getuser()` query
to read the clear-text (not PKBDF2) hex key string which it returns to Mosquitto
for authentication. If authentication passes, the connection is established.

For authorization, _auth_plug_ uses the identity as the username and the topic to
perform ACL-checking as described earlier.

The following log-snippet serves as an illustration:

```
New connection from ::1 on port 8885.
|-- psk_key_get(hint1, ps1) from [mysql] finds PSK: 1
New client connected from ::1 as mosqpub/90759-tiggr.ww. (c1, k60).
Sending CONNACK to mosqpub/90759-tiggr.ww. (0)
|-- user ps1 was authenticated in back-end 0 (psk)
|--   mysql: topic_matches(x, x) == 1
|-- aclcheck(ps1, x, 2) AUTHORIZED=1 by psk
Received PUBLISH from mosqpub/90759-tiggr.ww. (d0, q0, r0, m0, 'x', ... (2 bytes))
Received DISCONNECT from mosqpub/90759-tiggr.ww.
```

## Requirements

* [hiredis], the Minimalistic C client for Redis
* OpenSSL (tested with 1.0.0c, but should work with earlier versions)
* A [Mosquitto] broker
* A [Redis] server
* MySQL
* [TinyCDB](http://www.corpit.ru/mjt/tinycdb.html) by Michael Tokarev (included in `contrib/`).

## Credits

* Uses `base64.[ch]` (and yes, I know OpenSSL has base64 routines, but no thanks). These files are
>  Copyright (c) 1995, 1996, 1997 Kungliga Tekniska Hgskolan (Royal Institute of Technology, Stockholm, Sweden).
* Uses [uthash][2] by Troy D. Hanson.


 [Mosquitto]: http://mosquitto.org
 [Redis]: http://redis.io
 [pbkdf2]: http://en.wikipedia.org/wiki/PBKDF2
 [1]: https://exyr.org/2011/hashing-passwords/
 [hiredis]: https://github.com/redis/hiredis
 [uthash]: http://troydhanson.github.io/uthash/
 [MongoDB connection string]: https://docs.mongodb.com/manual/reference/connection-string/

## Possibly related

 * [mosquitto_pyauth](https://github.com/mbachry/mosquitto_pyauth)
 * [mosquitto-auth-plugin-http](https://github.com/hadleyrich/mosquitto-auth-plugin-http)
 * [lua_auth_plugin](https://github.com/DenkiYagi/lua_auth_plugin)

## Press

 * [How to make Access Control Lists (ACL) work for Mosquitto MQTT Broker with Auth Plugin](http://my-classes.com/2015/02/05/acl-mosquitto-mqtt-broker-auth-plugin/)
