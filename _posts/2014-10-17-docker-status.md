---
title: Web App Health Checks in Docker
categories:
    - Code
tags:
    - Zooniverse
    - Docker
---

Over at [The Zooniverse](https://www.zooniverse.org/) we've been migrating most
of our apps to run under [Docker](https://www.docker.com/). I won't bother
explaining all the benefits of Docker, but we've found it to be a great way of
packaging and deploying applications, especially since we have a lot of legacy
code with conflicting requirements. Getting everything running in Docker has
been interesting, and we've encountered a few problems that needed to be solved
along the way. One issue we needed to solve involved monitoring.

<!-- PELICAN_END_SUMMARY -->

The whole of the Zooniverse is deployed on [AWS](https://aws.amazon.com/), and
we basically have a handful of load balancers sending traffic to EC2 instances
which run various apps deployed inside Docker. There's one container running
Nginx on each instance, which routes requests to the other containers. We use
autoscaling to manage capacity, and to terminate and replace any instances that
fail. The problem was that it was possible for one container to fail without the
ELB health check detecting it, because the Nginx container was fine.

I considered a few of the usual tools to solve this problem. There's
[Supervisor](http://supervisord.org), which most of our apps run under already,
but they don't always fail in a way that Supervisor can detect; i.e. the app is
still running but in a broken state.  [Monit](https://mmonit.com/monit/) might
have worked, but would have taken effort to configure and fit in with our
deployment process. I wanted something lightweight that doesn't require any
effort to configure. It doesn't need to do anything fancy – it doesn't need
alerting, and it doesn't need to try to bring services back up when they fail.
It just needs to make the EC2 instance fail the ELB's health check when there's
a problem so that autoscaling can do its thing.

To that end I threw together an app that I've called
[docker-status](https://github.com/zooniverse/docker-status) (because that's how
creative I am with names). This is a simple [Flask](http://flask.pocoo.org) app
that finds any containers linked to it which expose port 80, asynchronously
checks each of them every 30 seconds to see what status code they're returning,
and responds to HTTP requests with a 200 if they're all OK or 500 otherwise.

That's all there is to it. There's nothing to configure – just link your web
app containers to it, make sure it's the default server in Nginx, and you're
done.

It's also available on the Docker Hub at
[zooniverse/docker-status](https://registry.hub.docker.com/u/zooniverse/docker-status/).
