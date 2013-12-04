

DEVELOPMENT BRANCH
==================

Calamari 2.0
============

Check out the `Architecture document`_ to get an idea of the overall
structure.

.. _Architecture document: https://docs.google.com/document/d/11Sq5UW3ZzeTwPBk3hPbrPI002ScycZQOzXPev7ixJPU/edit?usp=sharing


This code is meant to be runnable in two ways: in *production mode*
where it is installed systemwide from packages, or in *development mode*
where it us running out of a git repo somewhere in your home directory.

Installing in production mode is not described here because the short
version is "follow the same instructions we give to users".  Installing
in development mode is intrinsically a bit custom, so here are some
pointers:


Installing dependencies
-----------------------

If you haven't already, install some build dependencies (apt, yum, brew, whatever) which
will be needed by the pip packages:

::

    sudo apt-get install python-virtualenv git python-dev swig libzmq-dev g++

If you're on ubuntu, there are a couple packages that are easier to install with apt
than with pip (and because of `m2crypto weirdness`_)

::

    sudo apt-get install python-cairo python-m2crypto

1. Create a virtualenv
3. Install dependencies with ``pip -r requirements.txt``.
4. Install graphite and carbon, which require some special command lines:

::

    pip install carbon --install-option="--prefix=$VIRTUAL_ENV" --install-option="--install-lib=$VIRTUAL_ENV/lib/python2.7/site-packages"
    pip install graphite-web --install-option="--prefix=$VIRTUAL_ENV" --install-option="--install-lib=$VIRTUAL_ENV/lib/python2.7/site-packages"


5. Grab the `GUI code <https://github.com/inktankstorage/clients>`_, build it and
   place the build products in ``webapp/content`` so that when it's installed you
   have a ``webapp/content`` directory containing ``admin``, ``dashboard`` and ``login``.

.. _m2crypto weirdness: http://blog.rectalogic.com/2013/11/installing-m2crypto-in-python.html

*Aside: unless you like waiting around, set PIP_DOWNLOAD_CACHE in your environment*

Getting ready to run
--------------------

Calamari server consists of multiple modules, link them into your virtualenv:

::

    pushd rest-api ; python setup.py develop ; popd
    pushd cthulhu ; python setup.py develop ; popd
    pushd minion-sim ; python setup.py develop ; popd
    pushd calamari-web ; python setup.py develop ; popd

``carbon`` needs its configuration files:

::

    cp ${VIRTUAL_ENV}/conf/carbon.conf.example ${VIRTUAL_ENV}/conf/carbon.conf
    cp ${VIRTUAL_ENV}/conf/storage-schemas.conf.example ${VIRTUAL_ENV}/conf/storage-schemas.conf

You may want to increase carbon.conf:MAX_CREATES_PER_MINUTE for nice snappy setup (we also
do this in the production installer).

Graphite needs some folders created:

::

    mkdir -p ${VIRTUAL_ENV}/storage/log/webapp
    mkdir -p ${VIRTUAL_ENV}/storage/index

``salt-master`` needs is configuration files.  ``salt/etc/salt/master`` already exists
in the git repo, but unfortunately contains some absolute paths that **you'll need to edit**
each time you set up a repo.  Also, set the ``user`` variable to your username.

Create the REST API's database:

::

    pushd webapp/calamari ; python manage.py syncdb ; popd

Create the cthulhu service's database:

::

    calamari-ctl initialize


Running the server
------------------

The server processes are run for you by ``supervisord``.  A healthy startup looks like this:

::

    calamari john$ supervisord -n -c ./supervisord.conf
    2013-12-02 10:26:51,922 INFO RPC interface 'supervisor' initialized
    2013-12-02 10:26:51,922 CRIT Server 'inet_http_server' running without any HTTP authentication checking
    2013-12-02 10:26:51,923 INFO supervisord started with pid 31453
    2013-12-02 10:26:52,925 INFO spawned: 'salt-master' with pid 31456
    2013-12-02 10:26:52,927 INFO spawned: 'carbon-cache' with pid 31457
    2013-12-02 10:26:52,928 INFO spawned: 'calamari-frontend' with pid 31458
    2013-12-02 10:26:52,930 INFO spawned: 'cthulhu' with pid 31459
    2013-12-02 10:26:54,435 INFO success: salt-master entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: carbon-cache entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: calamari-frontend entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: cthulhu entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)

Supervisor will print complaints if something is not starting up properly.  Check in the various \*.log files to
find out why something is broken, or run processes individually by hand to debug them (see the commands in supervisord.conf).

At this point you should have a server up and running at ``http://localhost:8000/`` and
be able to log in to the UI.

Ceph servers
------------

Simulated minions
_________________

Impersonate some Ceph servers with the minion simulator:

::

    minion-sim --count=3




Real minions
____________

If you have a real live Ceph cluster, install ``salt-minion`` on each of the
servers, and configure it to point to your development instance host (mine is 192.168.0.5,
**substitute yours**)

::

    wget -O - https://raw.github.com/saltstack/salt-bootstrap/develop/bootstrap-salt.sh
    | sudo sh && echo "master: 192.168.0.5" >> /etc/salt/minion && service
    salt-minion restart

Allowing minions to join
________________________

Authorize the simulated salt minions to connect to the calamari server:

::

    salt-key -c salt/etc/salt -L
    salt-key -c salt/etc/salt -A

You should see some debug logging in cthulhu.log, and if you visit /api/v1/cluster in your browser
a Ceph cluster should be appear.

Further reading (including running tests)
-----------------------------------------

Build the docs:

::
    cd docs/
    make html
    open _build/html/index.html