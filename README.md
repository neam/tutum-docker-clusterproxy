tutum-docker-clusterproxy
=========================

HAProxy image that balances between linked containers and, if launched in Tutum, 
reconfigures itself when a linked cluster member joins or leaves


Usage
-----

Launch your application container that exposes port 80:

    docker run -d --name web1 tutum/hello-world
    docker run -d --name web2 tutum/hello-world

Then, run `tutum/haproxy` linking it to the target containers:

    docker run -d -p 80:80 --link web1:web1 --link web2:web2 tutum/haproxy

The `tutum/haproxy` container will listen in port 80 and forward requests to both `web1` and `web2` backends using a `roundrobin` algorithm.


Configuration
-------------

You can overwrite the following HAProxy configuration options:

* `BACKEND_PORT` (default: `80`): The port where the web application backends are listening to.
* `BACKEND_PORTS` (default **None**): The list of ports(comma separated) where the web application backends are listening to.(On works on Tutum at the moment)
* `FRONTEND_PORT` (default: `80`): The port where the load balancer is listening to.
* `MODE` (default: `http`): Mode of load balancing for HAProxy. Possible values include: `http`, `tcp`, `health`.
* `HDR` (default: `hdr`): "hdr" criteria in acl used in virtualhost. If set to `hdr_end`, for instance, haproxy will match all the subdomains'.
* `BALANCE` (default: `roundrobin`): Load balancing algorithm to use. Possible values include: `roundrobin`, `static-rr`, `source`, `leastconn`.
* `MAXCONN` (default: `4096`): Sets the maximum per-process number of concurrent connections.
* `OPTION` (default: `redispatch`): Comma-separated list of HAProxy `option` entries to the `default` section.
* `TIMEOUT` (default: `connect 5000,client 50000,server 50000`): Comma-separated list of HAProxy `timeout` entries to the `default` section.
* `SSL_CERT` (default: `**None**`): An optional certificate to use on the binded port. It should have both the private and public keys content. If set, port 443 will be used to handle HTTPS requests.
* `SSL_BIND_OPTIONS` (default: `no-sslv3`): Optional. Explicitly set which SSL bind options will be used for the SSL server. This sets the HAProxy `ssl-default-bind-options` configuration setting. The default will allow only TLSv1.0+ to be used on the SSL server.
* `SSL_BIND_CIPHERS` (default: `None`): Optional. Explicitly set which SSL ciphers will be used for the SSL server. This sets the HAProxy `ssl-default-bind-ciphers` configuration setting.
* `VIRTUAL_HOST` (default: `**None**`): Optional. Let HAProxy route by domain name. Format `LINK_ALIAS=DOMAIN`, comma separated.
* `STATS_PORT` (default: `1936`): Port for the haproxy stats section. If this port is published, stats can be accessed at `http://<host-ip>:<STATS_PORT>/`
* `STATS_AUTH` (default: `stats:stats`): Username and password required to access the haproxy stats.

Check [the HAProxy configuration manual](http://haproxy.1wt.eu/download/1.4/doc/configuration.txt) for more information on the above.


Usage within Tutum
------------------

Launch the service you want to load-balance using Tutum.

Then, launch the load balancer. To do this, select "Jumpstarts", "Proxies" and select `tutum/haproxy`. During the "Environment variables" step of the wizard, link to the service created earlier (the name of the link is not important), and add "Full Access" API role (this will allow HAProxy to be updated dynamically by querying Tutum's API). 

That's it - the proxy container will start querying Tutum's API for an updated list of containers in the service and reconfigure itself automatically.


Use case scenarios
------------------

#### My webapp container exposes port 8080, and I want the proxy to listen in port 80

Use the following:

    docker run -d --link webapp:webapp -e BACKEND_PORT=8080 -p 80:80 tutum/haproxy

#### My webapp container exposes port 80, and I want the proxy to listen in port 8080

Use the following:

    docker run -d --link webapp:webapp -e FRONTEND_PORT=8080 -p 8080:8080 tutum/haproxy

#### I want the proxy to terminate SSL connections and forward plain HTTP requests to my webapp to port 80

Use the following:

    docker run -d --link webapp:webapp -p 443:443 -e SSL_CERT="YOUR_CERT_TEXT" tutum/haproxy

The certificate in `YOUR_CERT_TEXT` is a combination of public certificate and private key. Remember to put `\n` between each line of the certificate. A way to do this, assuming that your certificate is stored in `~/cert.pem`, is running the following:

    docker run -d --link webapp:webapp -p 443:443 -e SSL_CERT="$(awk 1 ORS='\\n' ~/cert.pem)" tutum/haproxy

#### I want the proxy to terminate SSL connections and forward plain HTTP requests to my webapp to port 8080

Use the following:

    docker run -d --link webapp:webapp -p 443:443 -e SSL_CERT="YOUR_CERT_TEXT" -e BACKEND_PORT=8080 tutum/haproxy

#### I want to use SSL and redirect non-SSL requests to the SSL endpoint

Use the following:

    docker run -d --link webapp:webapp -p 443:443 -p 80:80 -e SSL_CERT="YOUR_CERT_TEXT" tutum/haproxy

#### I want to set up virtual host routing by domain

There are two ways to configure virtual hosts with this image.

**Method 1: configuring the proxy**

Example:

    docker run -d --name webapp1 tutum/hello-world
    docker run -d --name webapp2 tutum/hello-world
    docker run -d --link webapp1:webapp1 --link webapp2:webapp2 -e VIRTUAL_HOST="webapp1=www.webapp1.com, webapp2=www.webapp2.com" -p 80:80 tutum/haproxy

Notice that the format of `VIRTUAL_HOST` is `LINK_ALIAS=DOMAIN`, where `LINK_ALIAS` must match the *beginning* of the link name and `DOMAIN` is the HTTP host that you want the proxy to use to forward requests to that backend.

In the example above, when you access `http://www.webapp1.com`, it will show the service running in container `webapp1`, and `http://www.webapp2.com` will go to container `webapp2`.

If you use the following:

    docker run -d --name webapp1 tutum/hello-world
    docker run -d --name webapp2-1 tutum/hello-world
    docker run -d --name webapp2-2 tutum/hello-world
    docker run -d --link webapp1:webapp1 --link webapp2-1:webapp2-1 --link webapp2-2:webapp2-2 -e VIRTUAL_HOST="webapp1=www.webapp1.com, webapp2=www.webapp2.com" -p 80:80 tutum/haproxy

When you access `http://www.webapp1.com`, it will show the service running in container `webapp1`, and `http://www.webapp2.com` will go to both containers `webapp2-1` and `webapp2-2` using round robin (or whatever is configured in `BALANCE`).


**Method 2: configuring the webapp backends**

Alternatively, virtual hosts can be configured by the proxy reading linked container environment variables (`VIRTUAL_HOST`). Here is an example:

    docker run -d -e VIRTUAL_HOST=www.webapp1.com --name webapp1 tutum/hello-world
    docker run -d -e VIRTUAL_HOST=www.webapp2.com --name webapp2 tutum/hello-world 
    docker run -d --link webapp1:webapp1 --link webapp2:webapp2 -p 80:80 tutum/haproxy

In the example above, when you access `http://www.webapp1.com`, it will show the service running in container `webapp1`, and `http://www.webapp2.com` will go to container `webapp2`.

If you use the following:

    docker run -d -e VIRTUAL_HOST=www.webapp1.com --name webapp1 tutum/hello-world
    docker run -d -e VIRTUAL_HOST=www.webapp2.com --name webapp2-1 tutum/hello-world
    docker run -d -e VIRTUAL_HOST=www.webapp2.com --name webapp2-2 tutum/hello-world
    docker run -d --link webapp1:webapp1 --link webapp2-1:webapp2-1 --link webapp2-2:webapp2-2 -p 80:80 tutum/haproxy

When you access `http://www.webapp1.com`, it will show the service running in container `webapp1`, and `http://www.webapp2.com` will go to both containers `webapp2-1` and `webapp2-2` using round robin (or whatever is configured in `BALANCE`).

#### I want all my `*.node.io` domains point to my service

    docker run -d -e VIRTUAL_HOST=node.io --name webapp tutum/hello-world
    docker run -d --link webapp:webapp -e HDR="hdr_end" -p 80:80 tutum/haproxy


Topologies using virtual hosts
------------------------------

Within Tutum:

                                                         |---- container 1
                                  |----- service 1 ----- |---- container 2
                                  |   (virtual host 1)   |---- container 3
    internet --- tutum/haproxy--- |
                                  |                      |---- container a
                                  |----- service 2 ----- |---- container b
                                      (virtual host 2)   |---- container c


Outside Tutum (any Docker server):

                                  |---- container 1 (virtual host 1)
                                  |---- container 2 (virtual host 1)
                                  |---- container 3 (virtual host 1)
    internet --- tutum/haproxy--- |
                                  |---- container a (virtual host 2)
                                  |---- container b (virtual host 2)
                                  |---- container c (virtual host 2)
