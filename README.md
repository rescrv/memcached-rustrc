Fragmented Services.
====================

That's what I call them.

Processes that should be instantiated once per combination of variables.  Not serverless.  That's too close to magic.
Or inetd.  Either way, I want to run one process per fragment to provide a clear separation between fragments.  OSes are
good at processes.

Fragmented services build on [`rc_conf`](https://crates.io/crates/rc_conf) to provide a rich, inheritance-oriented way
to build or configure services.  For example, a base, non-fragmented memcached service deploy starts with two pieces.
The first is the `rc.d` service stub.  It looks like this:

```
#!/usr/bin/env rcscript

DESCRIBE=A single memcached instance.
COMMAND=memcached -v
```

The format is intended to be self-describing.  The only two gotchas two beware are that this file should be executable
and the command should run in the foreground and not daemonize.

If we try to run this service as-is it won't run.  By default, every service is disabled.  To enable memcached, create
an `rc.conf` with the following:

```
memcached_ENABLED="YES"
```

This will cause [`rustrc`](https://crates.io/crates/rustrc) (the supervisor written on `rc_conf`) to keep the memcached
daemon running with some provisions for backing off during failure.

This isn't a very useful example as our end state, provides a starting point.  We may want to configure memcached and
going back to the `rc.d` file and redeploying it every time won't be fun.

Enter the first principle of `rc_conf`:  A separation between configuration and where the value gets used.  `rc.d`
scripts provide the "hooks" for the configuration by taking command line applications and binding them to `rc.conf`
files.  For example, if we wanted to configure the port and hostname for memcached, we modify the `rc.d` script to look
like:

```
COMMAND=memcached -v ${HOST:+-i ${HOST}} ${PORT:+-p ${PORT}}
```

This will pull the values of HOST and PORT from the `rc_conf` where running memcached.  But globals get messy fast.
Whose HOST; whose PORT?  This is where `rc_conf` comes in.  To set the hostname, add this to `rc.conf`:

```
memcached_HOST="memcached.example.org"
memcached_PORT="11211"
```

If either of these were absent, the "memcache_"-prefix-free versions would be substituted in their place.  So to have a
default host for all services that can be overridden, specify `HOST=default.example.org`.

## Aliasing

Before getting to fragmented services, we need one last detour:  Service aliasing.  To spin up a second memcached on
22122, we can add this to our `rc.conf`:

```
memcached_two_INHERIT="YES"
memcached_two_ALIASES="memcached"
memcached_two_PORT="22122"
```

An alias such as this one uses one `rc.d` stub to launch two memcached instances.  This is the essence of fragmented
services.

## Fragmenting Things

A fragmented service fundamentally has some set of key-value pairs associated with it.  For example, we could partition
by location and customer in a fictitious example.  Add this to `rc.conf`:

```
VALUES_METRO="metros.conf"
VALUES_CUSTOMER="customers.conf"
```

And then create the `metros.conf` and `customers.conf` files:

```
# metros.conf
Sjc=
Lax=
Jfk=
```

```
# customers.conf
CyberDyne=
PlanetExpress=
TyrellCorp=
```

By convention, values like these take camel case so that they are distinguishable.  Feel free to upper or lower case the
entire string.  Avoid underscores, though (although if you do there are ways to accommodate them---send me an email or
reach out on GitHub).

Lastly we define our fragmented service with an "AUTOGEN" declaration in `rc.conf`.

```
METRO_CUSTOMER_memcached_AUTOGEN="YES"
METRO_CUSTOMER_memcached_ENABLED="MANUAL"
METRO_CUSTOMER_memcached_ALIASES="memcached"
METRO_CUSTOMER_memcached_INHERIT="YES"
```

This will allow each instance of the service to be manually started, but will not restart on failure and will not
automatically restart any service.  Let's change it to automatically start memcached for Planet Express near JFK with a
new host name and port.

```
Jfk_PlanetExpress_memcached_ENABLED="YES"
Jfk_PlanetExpress_memcached_HOST="planetexpress.example.org"
Jfk_PlanetExpress_memcached_PORT="4242"
```

Let's make the hostname different for each fragmented service:

```
METRO_CUSTOMER_memcached_HOST="${CUSTOMER}.${METRO}.memcached.example.org"
```

This won't change anything for Planet Express's instance running in Jfk, but will provide a default for all other
instances.  When reloaded, rustrc will converge the services by restarting any service whose bindings are obsolete.

Lastly, we may want to carefully watch SkyNet in SJC:

```
Sjc_CyberDyne_memcached_WRAPPER='strace -o cyberdyne.txt -f'
```

## Project Status

rustrc is a single-host program.  I'd love to make it distributed and compile down to something like Kubernetes, Helm,
Terraform, or any number of other systems as needed.  I'd especially like to pay attention to:

- Keeping it true to what's outlined here.  Cascades and inherited values are easy to reason about, easy to describe,
  and provide a decoupling between an application and its environment.
- Keeping the code simple.  There's less code than in some YAML parsers.
- Add guards to ensure that changes to VALUES_ or other parameters roll out in a safe way.

Want to play with it yourself?  Grab `rustrc` and `rc_conf` with `cargo install rustrc rc_conf` and clone this repo:

```
rcinvoke Sjc_CyberDyne_memcached
```
