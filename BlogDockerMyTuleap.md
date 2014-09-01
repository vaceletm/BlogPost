Deploy a PHP app with Docker, Nginx, FPM and CentOS SCL
=======================================================

Before going further on how we are building Tuleap Docker images, let's make
a quick stop on how we built the landing page for [Tuleap Cloud](http://mytuleap.com).

It's beautiful landing page made by Ben & Manon with fancy colors and mountains.
The technical part is as simple as possible, a set of assets (HTML, JS, images)
and a php simple script for the backend stuff (everythins is done asynchronously).

This time we decided to use Docker The Right Way: only element per container,
no supervisord !

The web container - Nginx & Software Collections
------------------------------------------------

The first container, the easiest one is the web front end.

As we are going to serve only statics elements, we use nginx. But as there is 
no reason that the only hipster in the team would be the FrontEnd developer, 
we want to use the latext nginx version (1.4).

Of course, this version is not available in default centos repositories. But
there are [SoftwareCollections](http://softwarecollections.org). Yay, there is
nginx-nginx14 package for us.

SoftwareCollections is a Red Hat backed initiative to provide recent (I would 
rather say decent) versions of popular tools without impacting the base system.

Hereafter, the Dockerfile to get it installed and running

    # We stay with centos6 now, centos7 is next on the list
    FROM centos:centos6

    MAINTAINER Manuel Vacelet <manuel.vacelet@enalean.com>

    # This custom repo file just activate "centos-plus" repo otherwise you will
    # get error with selinux dependencies
    ADD CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo

    # Update the base system with latest patches
    RUN yum -y update && yum clean all

    # Prepare the ground to use software collections
    RUN yum -y install scl-utils && yum clean all

    # Install nginx
    RUN rpm -i https://www.softwarecollections.org/en/scls/rhscl/nginx14/epel-6-x86_64/download/rhscl-nginx14-epel-6-x86_64-1-2.noarch.rpm
    RUN yum -y install nginx14-nginx && yum clean all

    # Configure nginx
    ADD docker-web/nginx.conf /opt/rh/nginx14/root/etc/nginx/nginx.conf

    # Deploy the static assets
    ADD .  /var/www/

    EXPOSE 80

    # Launch nginx at start
    CMD ["/opt/rh/nginx14/root/sbin/nginx"]

And that's all !

In 10 lines, you get the latest CentOs with rock solid and supported nginx.

I think the best summary come from Ben (our Designer and Front End expert, 
the guy with the Mac in the office): "Wow, this is easy, even for me".

The php backend (PHP FPM and SCL again)
---------------------------------------

This is all fun and games but there is one piece missing: this only serve static
content and we need to execute php.

Enters the second container: PHP FPM executor.

The purpose of this container will be to execute the PHP content, it will be
reverse "proxed" by nginx (the fpm container will not be reachable from outside.

Here again, we decided to use SCL to get PHP 5.4 (because of fun and why not).
But this time, no need to install specific rpm, PHP 5.4 is available in official
SCL repositories.

So the Dockerfile looks like:

    FROM centos:centos6

    MAINTAINER Manuel Vacelet <manuel.vacelet@enalean.com>

    ADD CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo
    RUN yum -y update && yum clean all
    RUN yum -y install scl-utils && yum clean all

    # Get the SCL repositories definition (.repo)
    RUN yum install -y centos-release-SCL && yum clean all

    # Install php54
    RUN yum -y install php54-php-fpm && yum clean all

    # Configure FPM
    RUN sed -i '/^listen = /clisten = 0.0.0.0:9000' /opt/rh/php54/root/etc/php-fpm.d/www.conf
    RUN sed -i '/^listen.allowed_clients/c;listen.allowed_clients =' /opt/rh/php54/root/etc/php-fpm.d/www.conf
    RUN sed -i '/^;catch_workers_output/ccatch_workers_output = yes' /opt/rh/php54/root/etc/php-fpm.d/www.conf

    ADD . /var/www/

    EXPOSE 9000

    # Move to the directory were the php files stands
    WORKDIR /var/www
    CMD ["/opt/rh/php54/root/usr/sbin/php-fpm"]

That's all for the fpm container.

Make the two containers work together
-------------------------------------

We need to update the web container and configure nginx to properly serve
the various contents. The final nginx conf:

    upstream nginx_backend {
        server %fpm-ip%:9000;
    }

    server {
        listen       80;
        server_name  mytuleap.com;

        location ~* \.(html|jpg|jpeg|gif|png|css|js|ico|xml)$ {
            root              /var/www;
            access_log        off;
            log_not_found     off;
            expires           360d;
        }

        location ~* \.php$ {
            root /var/www;
            include fastcgi.conf;
            fastcgi_pass nginx_backend;
        }
    }

You can notice in the upstream definition a placeholder %fpm-ip%. It's here 
because we cannot "pre-wire" the two images as we cannot predict the IP address
of the fpm container when we build the web image.

To address this point, we make use of Docker --link facilities. This option will
wire two containers and automatically expose the IP and Ports between containers

Example of env command when I start web container linked with a fpm container:

    $> docker run -d --name=fpm mytuleap.com/fpm
    $> docker run -ti --name=web --link fpm:fpm mytuleap.com/webfront env
    HOME=/
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    HOSTNAME=0c7b823a9c4e
    TERM=xterm
    FPM_PORT=tcp://172.17.0.20:9000
    FPM_PORT_9000_TCP=tcp://172.17.0.20:9000
    FPM_PORT_9000_TCP_ADDR=172.17.0.20
    FPM_PORT_9000_TCP_PORT=9000
    FPM_PORT_9000_TCP_PROTO=tcp
    FPM_NAME=/web/fpm

FPM_* variables are automatically exported by docker with --link, so we just
have to use that to replace our nginx placeholder. So, in web image, instead of
running nginx directly, we run this small script:

    #!/bin/bash

    sed -i "s/%fpm-ip%/$FPM_PORT_9000_TCP_ADDR/" /opt/rh/nginx14/root/etc/nginx/nginx.conf

    exec /opt/rh/nginx14/root/sbin/nginx

Send emails - ssmtp and Rackspace Mailgun
-----------------------------------------

We are almost done, the only missing part is that our PHP app should be able to
send emails. As we don't want to run our own mail server, we can rely on great
rackspace emailing service Mailgun. 
As usual with rackspace, it's "emails as if you meant it": simple, fast, easy 
and... it's free up to 50'000 emails / month account.

To send emails, we choose to use ssmtp. It's available in EPEL so in fpm 
Dockerfile we add:

    RUN rpm -ivh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    RUN yum -y install ssmtp && yum clean all
    ADD ssmtp.conf /etc/ssmtp.conf

    # apache needs to be in mail group so fpm can send emails
    RUN usermod -G mail apache

    # fpm needs to be aware that we are using ssmtp
    RUN sed -i '/^sendmail_path = /csendmail_path = \/usr\/sbin\/ssmtp -t' /opt/rh/php54/root/etc/php.ini

And we are done !

Wrap up
-------

As you can see with docker, centos and SCL you can get recent version of tools,
the deployment is made damn easy with docker and the configuration is simple.

Final note: instead of doing pattern matching with environnement variables to 
wire fpm and web containers, we could have used a service discovery (like [Consul](http://consul.io)). This will be the subject of a futur post, stay tuned !

In the meanwhile, you can [request your Tuleap](http://mytuleap.com) ;)
