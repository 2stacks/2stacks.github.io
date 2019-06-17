---
title: "Rapid Deployment of Web Apps with Docker"
excerpt:: "Create a load balanced web app with NGINX and Docker."
categories:
  - Github
tags:
  - Docker
  - NGINX
link: https://www.2stacks.net/docker-nginx-lb/
---

I recently started learning Docker and was immediately amazed by the [docker-compose][0] feature.  I've created a sample web application using [NGINX][2] that can be rapidly deployed using [docker-compose][5].  My GitHub repo page walks you through running and testing the application.  It is composed of an [NGINX loadbalancer][3] fronting two [NGINX web servers][4].  The demo can be downloaded at [2stacks/docker-nginx-lb][1].

[0]: https://docs.docker.com/compose/overview/
[1]: https://github.com/2stacks/docker-nginx-lb
[2]: http://nginx.org/en/
[3]: https://www.nginx.com/resources/admin-guide/tcp-load-balancing/
[4]: https://hub.docker.com/_/nginx/
[5]: https://docs.docker.com/compose/overview/