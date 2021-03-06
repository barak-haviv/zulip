# Deployment options

The default Zulip installation instructions will install a complete
Zulip server, with all of the services it needs, on a single machine.

For production deployment, however, it's common to want to do
something more complicated.  This page documents the options for doing so.

## Running Zulip's service dependencies on different machines

Zulip has full support for each top-level service living on its own
machine.

You can configure remote servers for Postgres, RabbitMQ, Redis,
in `/etc/zulip/settings.py`; just search for the service name in that
file and you'll find inline documentation in comments for how to
configure it.

Since some of these services require some configuration on the node
itself (e.g. installing our `postgres` extensions), we have designed
the puppet configuration that Zulip uses for installing and upgrading
configuration to be completely modular.

For example, you can install a Zulip rabbitmq server on a machine, you
can do the following after unpacking a Zulip production release
tarball:

```
env PUPPET_CLASSES=zulip::redis ./scripts/setup/install
```

You can see most likely manifests you might want to choose in the list
of includes in
[the main manifest for the default all-in-one Zulip server][voyager.pp],
though it's also possible to subclass some of the lower-level
manifests defined in that directory if you want to customize.  A good
example of doing this is in the
[zulip_ops puppet configuration][zulipchat-puppet] that we use as part
of managing chat.zulip.org and zulipchat.com.

### Using Zulip with Amazon RDS as the database

Unfortunately, you cannot use most third-party database-as-a-service
provides like Amazon RDS as the database provider with Zulip without a
degraded experience.  Zulip let you choose one of two
[full-text search postgres extensions](../subsystems/full-text-search.html).
Neither is available in Amazon RDS.  As a result, if you use one of
those providers, Zulip's full-text search will be unavailable.

## Putting the Zulip application behind a reverse proxy

If you'd like to run Zulip behind a reverse proxy server, you need to
make sure that your proxy server is configured in a way that is
compatible with Zulip's use of long-polling for real-time push from
the server to your users' browsers.

With an `nginx` reverse proxy, the key configuration options are, for
the `/json/events` and `/api/1/events` endpoints:

* `proxy_read_timeout 1200;`.  It's critical that this be
  significantly above 60s, but the precise value isn't important.
* `proxy_buffering off`.  If you don't do this, your `nginx` proxy may
  return occasional 502 errors to clients using Zulip's events API.

The other tricky failure mode with `nginx` reverse proxies is that
they can load-balance between the IPv4 and IPv6 addresses for a given
hostname.  This can result in mysterious errors that can be quite
difficult to debug.  Be sure to declare your `upstreams` in a way that
won't do load-balancing unexpectedly (e.g. pointing to a DNS name that
you haven't configured with multiple IPs for your Zulip machine;
sometimes this happens with IPv6 configuration).

You can look at our
[nginx reverse proxy configuration][nginx-loadbalancer] to see an
example of how to do this properly (the various include files are
available via the `zulip::nginx` puppet module).

[voyager.pp]: https://github.com/zulip/zulip/blob/master/puppet/zulip/manifests/voyager.pp
[zulipchat-puppet]: https://github.com/zulip/zulip/tree/master/puppet/zulip_ops/manifests
[nginx-loadbalancer]: https://github.com/zulip/zulip/blob/master/puppet/zulip_ops/files/nginx/sites-available/loadbalancer

