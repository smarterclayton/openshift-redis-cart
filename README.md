OpenShift Redis Cartridge
=========================

Runs [Redis](http://redis.io) on [OpenShift](https://openshift.redhat.com/app/login) using downloadable cartridge support.  To install to OpenShift from the CLI (you'll need version 1.9 or later of rhc), create your app and then run:

    rhc add-cartridge http://cartreflect-claytondev.rhcloud.com/reflect?github=smarterclayton/openshift-redis-cart

Any log output will be generated to $OPENSHIFT_REDIS_DIR/logs/redis.log

Supports clustering and persistence, with monitoring from Redis Sentinel.  Does not automatically pull the latest security updates - please monitor the Redis upstream 2.6 branches.


How it Works
------------

`conf/redis.conf.erb` is run before Redis starts and generates a configuration file.  It will attempt to load the file `.openshift/redis.conf` if it exists.  The default configuration will snapshot the db to disk into the <code>$OPENSHIFT_DATA_DIR/.redis/dbs</code> directory.

At cart creation, a unique password will be generated to the environment variable `REDIS_PASSWORD` - the server will require this to connect.

To access `redis-cli` from the SSH session, use the $REDIS_CLI environment variable to get the correct config on the gear

    $ ssh <gear with redis>
    Connectiing to....
    $ redis-cli $REDIS_CLI
    127.0.32.124 6379> 

To connect to Redis from your local shell, run the <code>cartridge-status</code> command to get the port and authorization info:

    $ rhc cartridge-status redis -a <yourapp>

    RESULT:
    Redis is running
      master (receives writes), mode sharded
      Connect to: 343928-abc.dev.rhcloud.com:35546 password: 80o9euk80oeu90834

and then run <code>rhc port-forward</code> to your app:

    $ rhc port-forward <yourapp>
    ....
    redis   127.0.0.1:35546   => 343928-abc.dev.rhcloud.com:35546

In another shell window, run <code>redis-cli</code>

    $ redis-cli -p 35546 -a 80o9euk80oeu90834
    redis 127.0.0.1:35546>

A number of Redis configuration values can be tuned via environment variables:

*  <code>REDIS_PASSWORD</code>

   The password for accessing Redis.

*  <code>REDIS_MAXMEMORY</code>

   The maximum memory this instance will allow.  No default value.

*  <code>REDIS_APPENDONLY</code>

   The value of <code>appendonly</code> in redis.conf.  Defaults to 'no'

*  <code>REDIS_APPENDFSYNC</code>

   The value of <code>appendfsync</code> in redis.conf.  Defaults to 'everysec'

Always restart each gear after setting these environment variables.


Upgrading
---------

If you install this cartridge from source, you will be using a precompiled version of Redis 2.6 for RHEL6.  You can run the <code>bin/control update</code> script on each gear to build and update to the latest version of the Redis 2.6 tree.  

    $ rhc ssh <yourapp> --gears 'cd redis && ./bin/control update'
    $ rhc restart-cartridge redis -a <yourapp>'

We hope to add a more natural update process at a later point.


Redis Clustering
----------------

The Redis cartridge supports two modes:

*  Sharded, with no slaves

   Each scaled gear you add is a new master - you can talk to each, but there is no slave configuration created.  Clients are responsible for determining which client to talk to.

*  Read replicas, one master

   Each scaled gear you add after the first becomes a slave in read only mode - changes you make to the master are automatically propagated.

At this time, when you scale up Redis the master host and port are not automatically communicated to your web gears - you'll need to set your own environment variables on each gear or in your client code.  In the future, we'll support exposing an environment variable that contains the hosts and ports of each member of the cluster.

Setting the mode requires an environment variable to set on each gear.  Until application wide environment variables are supported, you must set the variable on each Redis gear, then send the scale-cartridge command to instruct Redis to recalculate its internal mode.

    # determine the size of your cluster
    $ rhc scale-cartridge redis -a <yourapp> 2

    # environment variables are not copied on scale up today
    $ rhc ssh <yourapp> --gears 'cd redis && echo read_replica > env/REDIS_DB_MODE'

    # recalculate the mode
    $ rhc scale-cartridge redis -a <yourapp> 2

    # have the mode take effect
    $ rhc restart-cartridge redis -a <yourapp>

Scaling the cartridge recalculates the configuration, and then restart makes the configuration change take effect.  Run <code>rhc cartridge-status</code> to see the outcome of your change:

    $ rhc cartridge-status redis -a foo
    RESULT:
    Redis is running
      slave
      Connect to: gear1-abc.dev.rhcloud.com:35541 password: a_pass
    Redis is running
      master (receives writes), mode read_replica
      Connect to: gear1-abc.dev.rhcloud.com:35546 password: a_pass

Note that in scaled apps, port forward may not correctly represent the correct master.  You may need to port forward directly to the scaled gears via <code>ssh</code>.


Redis Sentinel
--------------

Redis Sentinel can be configured to run with the Redis cartridge and provide master failover and monitoring in the 'read_replica' cluster mode.  Each gear will run one instance of Sentinel, and if the master is detected to be down one of the slave gears will be elected to be the new master.  Your clients must be aware of the possibility of failover.

To enable Sentinel support, set the <code>REDIS_SENTINEL</code> environment variable in each gear to <code>1</code>.

    $ rhc ssh <yourapp> --gears 'cd redis && echo 1 > env/REDIS_SENTINEL'
    $ rhc restart-cartridge redis -a <yourapp>

Run <code>rhc cartridge-status</code> to get the connection information for Sentinel.

By default, the quorum is <code># of gears / 2 + 1</code>.  If you want to run two gears, set the environment variable <code>REDIS_SENTINEL_QUORUM</code> to <code>1</code> on each gear to instruct Sentinel to perform failover to the slave.


Future improvements
-------------------

* Get support for environment variable export from a connection hook (to publish to the web gears)
* Investigate running in plugin mode and in scaled cluster mode
* Add additional slave modes.