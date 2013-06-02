OpenShift Redis Cartridge
=========================

Runs [Redis](http://redis.io) on [OpenShift](https://openshift.redhat.com/app/login) using downloadable cartridge support.  To install to OpenShift from the CLI (you'll need version 1.9 or later of rhc), create your app and then run:

    rhc add-cartridge http://cartreflect-claytondev.rhcloud.com/reflect?github=smarterclayton/openshift-redis-cart

Any log output will be generated to $OPENSHIFT_REDIS_DIR/logs/redis.log


How it Works
------------

`conf/redis.conf.erb` is run before Redis starts and generates a configuration file.  It will attempt to load the file `.openshift/redis.conf` if it exists.  The default configuration will snapshot the db to disk.

At cart creation, a unique password will be generated to the environment variable `REDIS_PASSWORD` - the server will require this to connect.

To access `redis-cli` from the SSH session, use the $REDIS_CLI environment variable to get the correct config

    redis-cli $REDIS_CLI


Future improvements
-------------------

* Ensure the cart works with scaled apps
* Do compilation of the redis CLI in a CDK build (add a build hook to download latest source)
* Figure out how to export REDIS_CLI to other gears
* Investigate running in plugin mode and in scaled cluster mode
* Support multiple persistence modes via some simple config mechanism
* Test shared nothing scaling and exporting environment variables for each IP address
* Test sentinel startup