# Onionize

A docker wrapper for [Tor](https://torproject.org) v3 onion services (hidden services). It uses [docker-gen](https://github.com/jwilder/docker-gen) to configure Tor automatically when other containers are connected to the same network. The advantage of this approach is that it allows for optional network isolation and doesn't require building any containers yourself.

## Usage
Onionize is a Docker container that automatically exposes other selected Docker containers as onion services.

To try it, just run the Onionize container:

	docker run --name onionize -d -v /var/run/docker.sock:/tmp/docker.sock:ro torservers/onionize

This will download the Onionize image and create a running container instance from it called `onionize`.

Now you'll want to create an isolated network that doesn't have access to the internet for your services to reside in:

	docker network create -o "com.docker.network.bridge.enable_ip_masquerade=false" faraday

This created a network called `faraday` that has IP masquerading disabled, meaning it won't be able to access the internet. Now attach `faraday` to the running `onionize` container:

	docker network connect faraday onionize

Now the `onionize` container is connected to two networks: the default docker bridge providing internet connectivity, and the isolated `faraday` network where we'll start our services. For the purposes of this tutorial, we'll use the stock nginx image just so it's easy to see it working. Start an nginx instance connected to the `faraday` network with a special environment variable:

	docker run -d --net faraday -e ONIONSERVICE_NAME=nginx nginx

Alternatively, if you are making your containers already available via clearweb and want to add an onion service, you can simply connect its network to the `onionize` container (don't forget to set the `ONIONSERVICE_NAME` environment variable). For example:

	docker network connect nextcloud_default onionize

The environment variable `ONIONSERVICE_NAME` is read by `docker-gen` running in the `onionize` container, which according to its template identifies it as something that should be added to the list of onion services in `torrc`. It automatically uses whichever port is exposed by the container by default; if there's more than one, it uses port 80 by default but that behavior can be overridden by specifying a `ONIONSERVICE_PORT` environment variable as well. Containers specifying the same `ONIONSERVICE_NAME` are added to the same service. In this way, you can have multiple different containers providing services on different ports of the same .onion address. To see if it worked and to find out what .onion address was assigned, you can execute the following command:

	docker exec onionize cat /var/lib/tor/onion_services/<ONIONSERVICE_NAME>/hostname

Where `<ONIONSERVICE_NAME>` is replaced with the name you provided in the environment variable. This should print out a long hash followed by .onion, for instance:

	7fa6xlti5joarlmkuhjaifa47ukgcwz6tfndgax45ocyn4rixm632jid.onion

Now if you open the [Tor browser](https://www.torproject.org/download/download-easy.html.en) and paste the .onion address into the address bar, you should see the default nginx page.

Awesome!

If you use the "faraday" method outlined above, the nginx container can't access the internet, so it's much harder for it to leak network data! One caveat for this method is that it will only work for services that do not need to connect out to the internet, in an anonymized fashion or otherwise. If you're running a service that needs access to the internet, you'll need to either configure your service so that it proxies its connection over Tor, or you can look into some experimental work on [a custom Tor network driver for Docker](https://github.com/jfrazelle/onion).

## Useful to debug

	docker logs onionize
	docker exec onionize cat /etc/torrc 
# Credits

This work is largely based on [jheretic's onionboat](https://github.com/jheretic/onionboat).

Changes:

 * replaced Debian jessie with alpine:latest
 * updated docker-gen version
 * got rid of the bash startup script
 * simplified Tor config template
 * use modern v3 onions
 * keep all Tor data persistent (guard config, cached descriptors etc), not just the onion service keys
 * remove option to publish SOCKS port. if you need a dockerized tor client, this is not the image you want.
