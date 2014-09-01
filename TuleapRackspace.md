How Tuleap uses Rackspace
=========================

Thanks to Rackspace Open Source program, Tuleap Community benefit of a voucher
for usage of Rackspace infrastructure.

Not only Rackspace offers to open source projects is generous but the infra and
the teams are awesome (fanatical support is not just as tag line, it means something) !

So, after a couple of weeks of setting up the things, how do we use our voucher ?

Standard hosting
----------------

First of all, the standard hosting:

* [Tuleap.net](http://tuleap.net)
* [Gerrit](http://gerrit.tuleap.net)

For our very own Tuleap instance nothing fancy, we just setup a simple "Performance1-4"
server, running 24/7 to host our git repositories, trackers and so on.

The good point here is: it just works as it should. Spawn of the server took a less
than 1mn, provisionning 5 minutes and boom! it's here.

Continuous integration, the tricky part
---------------------------------------

But what's awesome with cloud servers is that you can create yourself a ton of
new problems you would never imagine with a bare metal servers. This is what we
have done for our continuous integration.

[Yannis already wrote about our CI](http://www.tuleap.org/tuleap-continuous-integration-infrastructure) so
I will not re-write here how gerrit/jenkins work but just to say that we heavily
rely on continuous integration in our day to day work. We cannot commit if the
CI is broken, hence we need a very reliable infrastructure.

For each commit, we need to run tests harness in several environments:

* Unit tests in Centos 5 with php 5.1
* Unit tests in Centos 6 with php 5.3 (+ 5.4 and 5.5 in the coming weeks)
* REST end-to-end tests in Centos 6 with php 5.3

As of today, 3 jobs are triggered for each gerrit patchsets. Each commit means
we need to have very fast jobs (we don't want to wait for hours between 2 commits).

The good thing is that most of the work / commits are done during day (9am-7pm) in
European / Africa timezones. So we don't need to have build slaves up day and night
doing nothing.

We don't want to waste Rackspace gift so we configured our CI with:

* Jenkins master is just a thin instance (Performance-1... you know, Java stuff) running
  24/7.
* Build slaves are spawned "just in time" when a commit occurs, the slave last for
  30mn if there is nothing to build and is destroyed. If another job need the slave,
  it's reused. Here we use Performance1-4 for better IO throughput

Just in time slaves with JClouds
--------------------------------

[JClouds](https://wiki.jenkins-ci.org/display/JENKINS/JClouds+Plugin) is a Jenkins
plugin to interact with various cloud providers, including Rackspace.

The configuration is dead simple, you just need to create RSA public/private keys
and select the kind of images you want to spawn and the hardware you wish.

The magic is done in slave configuration:

* Number of Executors: the number of job a slave can handle in parallel (we set to 5)
* The hardware ID (performance1-4 in our case)
* The image ID and setup script, this need some details

### Image ID and Init script

This is were you do the slave configuration part. Here we choose a "start with fresh environment" move:

* The image ID is a default Centos image 6.5 (PVHVM for better IO)
* Then all the provisionning is done with chef to get a usable server (Init script)

Init script:

    curl -L https://www.opscode.com/chef/install.sh | bash
    yum install -y git
    git clone https://github.com/vaceletm/tuleapci-deploy-centos6.git
    cd tuleapci-deploy-centos6
    echo -nE '###private json config###' > node.json
    chef-solo -c solo.rb -j node.json

And that's all.
It means that, when a job need a slave (and there is none available), JCloud will
create a new Rackspace server with fresh Centos install. Then we customize it for
our needs with our [Chef recipes](https://github.com/vaceletm/tuleapci-chef-cookbook).

There is a little extra time to pay for that (instead of taking 3mn to run, the very
first job will take ~10/15 minutes) but it's worth it because we build in a 100%
under control environment and it's very easy to upgrade it at will (just update
chef recipes).

Future
------

We are already extremely grateful to Rackspace. The offer is amazing and the support
teams are really awesome. The few time we had issues (when you spawn ~10/20 servers
a day you are likely to hit problems at some stage) we got answers of what was going
on within minutes. I don't remember spending more than 15mn on chat without having
an explanation about what's going on and a solution.

The next move for us is to turn our JClouds slaves in docker slaves. More about
that in a future post!
