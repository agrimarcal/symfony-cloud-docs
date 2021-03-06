.symfony.cloud.yaml
===================

SymfonyCloud configuration is made of several files that need to be stored along
side the project's code. The main file is called ``.symfony.cloud.yaml`` and
must be stored in the application root directory. This file configures how the
application is built and then deployed on SymfonyCloud.

The best way to get started is to run ``symfony init``. This command detects
what technologies are used by a project and generates configuration files with
sensible defaults. This generation is made using `Open-Source templates
<https://github.com/symfonycorp/cloud-templates>`_. Below is an example for a
Symfony application using Flex:

.. code-block:: yaml

    # .symfony.cloud.yaml
    name: 'app'

    type: 'php:7.4'

    build:
        flavor: 'none'

    disk: 128

    runtime:
        extensions:
            - apcu
            - mbstring
            - pdo_pgsql

    relationships:
        database: "mydatabase:postgresql"
        elasticsearch: "mysearch:elasticsearch"

    mounts:
        '/var': { source: local, source_path: var }
        '/public/upload': { source: local, source_path: uploads }

    web:
        locations:
            '/':
                root: 'public'
                passthru: '/index.php'

            '/upload':
                root: 'public/upload'
                scripts: false
                allow: true

    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2 symfony-build)
        deploy: |
            set -x -e
            (>&2 symfony-deploy)

.. note::

    This file configures one application. If you have multiple applications
    inside your project, you need one ``.symfony.cloud.yaml`` file at the root
    directory of each application.

    See the :doc:`Multiple applications </cookbooks/multi-apps>` cookbook for
    more information.

.. _application-name:

Name
~~~~

A **unique identifier** for the application inside the project. Only lower case
alpha-numeric characters (``a-z0-9``) are allowed.

This name is used in the ``.symfony/routes.yaml`` file to define the HTTP
upstream or in the `relationships`_ section in another ``symfony.cloud.yaml``
within the same :doc:`multi-applications project </cookbooks/multi-apps>`.

.. caution::

    Changing the name of an app once it has been deployed **destroys all storage
    volumes**; this is like deleting it and starting from scratch.

Type
~~~~

The **type** value defines the application language and the version to use.
Current supported values are:

* :ref:`php </languages/php>`
* nodejs
* python
* ruby
* go
* java (alpha support)

Build
~~~~~

Defines the build flavor; use ``none`` for PHP projects.

.. _config-size:

Size
~~~~

By default, SymfonyCloud allocates CPU and memory resources to each container
automatically. Some services are optimized for high CPU load, some for high
memory load. By default, SymfonyCloud tries to allocate the largest "fair" size
possible to all services, given the available resources on the plan. That is not
always optimal, however, and you can customize that behavior on any application
container or on any service.

Legal values for the size key are ``AUTO`` (the default), ``S``, ``M``, ``L``,
and ``XL``.

.. note::

    Note that in a development environment this value is ignored and always set
    to ``S``. It will only take effect in a production deployment (a master
    branch with an associated domain).

.. caution::

    If the total resources requested by all apps and services is larger than
    what the plan size allows then a production deployment will fail with an
    error.

.. _config-disk:

Disk
~~~~

Defines the disk quota for the application, in MB. The minimum disk size is
128MB. The following are counted toward the application's disk quota:

* The application root directory (source code, assets, vendors, etc);
* Writable and persistent directories (see `Mounts`_);
* logs (each trimmed to 10 MB automatically)
* About 50MB for internal state and configuration

When an application reaches its quota, writes are not allowed anymore. Depending
on the application requirements, this might lead to errors.

.. _mounts:

Mounts
~~~~~~

By default, the application's disk is read-only. Some directories can be marked
as persistent and writable via the ``mounts`` section; their contents are shared
from one deployment to the next:

.. code-block:: yaml

    # .symfony.cloud.yaml
    mounts:
        '/public/upload': { source: local, source_path: uploads }

The keys represent the path relative to the application root directory where the
directory is exposed (``/public/upload`` in the example).

The ``source_path`` property is the name of a subdirectory exposed from the
global persistent storage (``uploads`` in our example).

Dependencies
~~~~~~~~~~~~

Specifies dependencies that the application needs during the :ref:`build
<build-hook>` process. These dependencies are independent of the application
dependencies, and are installed globally in the ``PATH`` before the :ref:`build
<build-hook>` process and are always available. SymfonyCloud supports pulling
dependencies for the following languages:

* PHP
* Python
* Ruby
* Node.js
* Java (with integrated Maven and Ant support)

.. code-block:: yaml

    # .symfony.cloud.yaml
    dependencies:
        php:
            symfony/var-dumper: "*"
        python:
            behave: "*"
        ruby:
            sass: "3.4.7"
        nodejs:
            grunt-cli: "~0.1.13"

Runtime
~~~~~~~

The ``runtime`` section customizes the runtime language. Possible values vary
greatly from one language to the other. Please refer to specific language
documentation:

* :doc:`PHP </languages/php>`

.. _relationships:

Relationships
~~~~~~~~~~~~~

Defines how :doc:`services </services/intro>` are exposed to the application:

.. code-block:: yaml

    # .symfony.cloud.yaml
    relationships:
        database: "mydatabase:postgresql"
        elasticsearch: "mysearch:elasticsearch"

A service exposed to an application is called a **relationship**. Services are
exposed to the application using environment variables. The key defines the name
of the relationship and the prefix of the environment variables. The value is
formed as ``<service>:<endpoint>``. ``service`` refers to the name of one
service as defined in ``.symfony/services.yaml`` and ``endpoint`` refers to the
available endpoints exposed by the service.

.. note::

    See the :doc:`Services documentation </services/intro>` for a complete
    configuration of currently supported service types, endpoints and associated
    environment variables.

Web
~~~

The ``web`` section defines how the application is publicly exposed.

The following subkeys are available:

* `locations`_
* `upstream`_
* `commands`_

Locations
.........

The ``locations`` section associates absolute URI prefixes (**location**) to
a specific configuration:

.. code-block:: yaml

    # .symfony.cloud.yaml
    web:
        locations:
            "/":
                root: "public"
                passthru: "/index.php"
                index:
                    - index.php
                expires: -1
                scripts: true
                allow: true
                rules:
                    '\.(gif|jpe?g|png)$':
                        expires: 60
            "/admin":
                expires: 300
                passthru: true
                allow: true

Here is the reference of ``location`` configurations:

* ``root`` defines the folder used to serve static content for the given
  location, relative to the application root. Typical values for this property
  are ``public`` or ``web``. Absolute paths are not supported.

  .. caution::

    SymfonyCloud requires that the location root must not be the root of the
    project. It is important for security that private file mounts are not
    web-accessible.

* ``expires``: Enables the ``Cache-Control`` and ``Expires`` headers for static
  resources. Can be a duration or ``-1`` for no caching (default). Times can be
  suffixed with "ms" (milliseconds), "s" (seconds), "m" (minutes), "h" (hours),
  "d" (days), "w" (weeks), "M" (months, 30d) or "y" (years, 365d).

* ``index`` determines the file (or files) to consider when serving a request
  for a directory. It can be a filename, an array of filenames, or ``null``.
  Typically ``index.php``.

* ``scripts`` allows or disallows execution of scripts in that location.
  Possible values are ``true`` or ``false``, default is ``true``.

* ``passthru`` makes requests forwarded to the application if the resource is
  missing. Typical values for non-PHP applications are ``true`` or ``false``. In a
  PHP application, the value is the path to the front controller such as
  ``/index.php`` or ``/app.php``. The default value is ``false``.

* ``headers`` defines any additional headers to apply to static assets. This
  section is a mapping of header names to header values. Responses sent by the
  application aren't affected.

* ``allow`` allows or disallows serving files which don't match a rule. Possible
  values are ``true`` or ``false``, default is ``true``.

* ``request_buffering`` allows to enable or disable request buffering. As most
  application servers do not support chunked requests (e.g. fpm, uwsgi),
  ``request_buffering`` is enabled by default.

* ``rules`` defines rules to override current settings for some requests. The
  key is a PCRE regular expression that is matched against the full request URI,
  content is the same structure as `locations`_ configuration.

Upstream
........

.. note::

    For PHP applications, this section is optional and generally do not need to
    be specified.

``upstream`` is an optional key that describes how the application listens to
requests, in particular which protocol is used:

.. code-block:: yaml

    # .symfony.cloud.yaml
    web:
        upstream:
            socket_family: tcp
            protocol: http

* ``socket_family`` describes whether the application listens on a Unix socket
  (``unix``) or a TCP one (``tcp``, the default). When defined to ``unix``,
  SymfonyCloud sets the ``SOCKET`` environment variable with the path to the
  socket that the application must listen to. With ``tcp``, SymfonyCloud
  sets the ``PORT`` environment variable with the port that the application must
  listen on.

  .. tip::

    If the application is not listening at the expected location, SymfonyCloud
    returns *502 Bad Gateway* errors.

* ``protocol`` specifies if the application expects to receive incoming requests
  over the HTTP or the FastCGI protocol. Possible values are ``http`` and
  ``fastcgi``. The default varies depending on the runtime the application is
  using.

Commands
........

.. note::

    For PHP applications, this section is optional and generally do not need to
    be specified.

Under the ``commands`` section, the ``start`` entry defines the command used to
launch the application server listening according to the `upstream`_
configuration:

.. code-block:: yaml

    # .symfony.cloud.yaml
    web:
        commands:
            start: "gunicorn -b $PORT project.wsgi:application"

Hooks
~~~~~

The ``hooks`` section defines the scripts that SymfonyCloud runs at specific
times of an application lifecycle, build, deploy and post-deploy:

.. code-block:: yaml

    # .symfony.cloud.yaml
    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2 symfony-build)
        deploy: |
            set -x -e

            (>&2 symfony-deploy)
        post_deploy: |
            symfony console app:index

.. caution::

    Each hook is executed as a single script, so they will be considered failed
    **only if the final command fails**. Starts the script with ``set -e`` to
    make them fail on the first failed command, this is the behavior of default
    templates.

Three hooks are available:

.. _build-hook:

* First, the **build** script runs when SymfonyCloud packages the application.
  At this time, the application source code is checked out, global
  `dependencies`_ are installed and the filesystem can be written at will but no
  :doc:`services </services/intro>` are available. This is the best time to run
  heavy-duty tasks than can be performed offline such as assets build or Symfony
  container compilation.

  .. caution::

    The *build* step creates a container image that is tied to the sha1 of
    the Git tree being built. This image is reused for all deployments and all
    environments, including production. This means that the build step **must**
    be environment agnostic.

.. _deploy-hook:

* Then, the **deploy** script runs when SymfonyCloud is deploying the
  application. During the execution of this script, services are available.
  However, the filesystem where the application lives is read-only, except for
  the ``mounts``. This is the perfect time to run tasks requiring services and
  essential for a successful deployment such as database migrations.

  .. caution::

    All incoming requests are held until the end of the execution of
    the *deploy* script. Make sure this script runs as fast as possible.

.. _post_deploy-hook:

* Finally, the **post_deploy** script runs once the application is deployed and
  started. Requests are flowing in and as for the *deploy* script, services are
  available but filesystem is read-only. This is the perfect time to run tasks
  that need to be performed online but are not essential for deployment, search
  engine indexation for example.

.. tip::

    ``symfony deploy`` streams the *build* logs only. Logs for the ``deploy``
    and ``post_deploy`` hooks are stored in ``/var/log/deploy.log``. Access them
    via ``symfony log deploy``.

To better understand the big picture and how those three steps articulate with
each other, we invite you to read about :ref:`building the application
<application-build>` and :ref:`deploying the application <application-deploy>`
in the :doc:`What is SymfonyCloud? </intro>` article.

.. tip::

    To execute some actions during the *deploy* or *post_deploy* hooks only for
    a specific environment (to do some anonymization for example), the simplest
    way is to use the ``SYMFONY_BRANCH`` environment variable in a condition:

    .. code-block:: yaml

        # .symfony.cloud.yaml
        hooks:
            deploy: |
                if [[ "$SYMFONY_BRANCH" != "master" ]]; then
                    symfony console app:dev:anonymize
                fi

curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
...............................................................

The **configurator** is a script specially crafted for SymfonyCloud.
It ensures that projects are always using the most up-to-date version of some
tools:

- `croncape <https://github.com/symfonycorp/croncape>`_
- `Symfony CLI <https://symfony.com/download>`_
- `Composer <https://getcomposer.org/download/>`_

Additionally, it creates four helpers: symfony-build_, symfony-deploy_,
symfony-database-migrate_ and yarn-install_.

.. _symfony-build:

(>&2 symfony-build)
...................

**symfony-build** is our recipe to build a Symfony application the best way
possible. It removes the development frontend, install the application
dependencies using Composer (and Yarn, by running `yarn-install`_), optimize the
autoloader, build Symfony cache if possible and finally build the production
assets using Encore.

Composer install is run using default flag values. If you need to override those
flags, you can define them using the ``$COMPOSER_FLAGS`` environment variable:

.. code-block:: yaml

    # .symfony.cloud.yaml
    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2 COMPOSER_FLAGS="--ignore-platform-reqs" symfony-build)

One can also set ``NO_YARN`` to any value to disable all Yarn and assets
automation during the symfony-build_ run.

.. _symfony-deploy:

(>&2 symfony-deploy)
....................

**symfony-deploy** is to be used each time a Symfony application is deployed.
Its purpose is to make Symfony cache used by the application if built by
symfony-build_, built it otherwise. When ran from the web container, it
also runs the symfony-database-migrate_ helper.

.. _symfony-database-migrate:

(>&2 symfony-database-migrate)
..............................

You usually don't need to worry about this helper.
**symfony-database-migrate** purpose is to run database migrations. By default,
it will run your Doctrine migrations if ``doctrine/doctrine-migrations-bundle``
is installed. If your application uses another migration system you can
override ``/app/.global/bin/symfony-database-migrate`` during build time and
symfony-deploy_ will make use of it. You can use this script at any moment if
you need to run migrations manually or if you need to run them for
:doc:`workers </cookbooks/workers>`.

yarn-install
............

**yarn-install** is a script that installs Node and Yarn the best way possible
in a PHP container on SymfonyCloud. Similarly to Composer install, you can
customize Node setup and Yarn install behaviors using the following environment
variables:

* ``NVM_DIR``: Directory used to install NVM and Node. Default value is
  ``/app/.nvm``.

* ``NODE_VERSION``: Pinpoint the Node version that NVM is going to install.
  Default is ``--lts``

* ``YARN_FLAGS``: Flags to pass to ``yarn install``. No value by default.

Shall you need to use the Node installation setup by symfony-build_, you can
use the following snippet:

.. code-block:: yaml

    # .symfony.cloud.yaml
    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2 symfony-build)
            (>&2
                # Setup everything to use the Node installation
                unset NPM_CONFIG_PREFIX
                export NVM_DIR=$HOME/.nvm
                set +x && . "${NVM_DIR}/nvm.sh" use --lts && set -x
                # Starting from here, everything is setup to use the same Node
                yarn encore dev
            )

Or if you want to use two different Node versions:

.. code-block:: yaml

    # .symfony.cloud.yaml
    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2 symfony-build)
            (>&2
                cd web/js_app
                unset NPM_CONFIG_PREFIX
                export NVM_DIR=$HOME/.nvm

                NODE_VERSION=8 yarn-install

                # Setup everything to use the Node installation
                set +x && . "${NVM_DIR}/nvm.sh" use 8 && set -x

                # Starting from here, everything is setup to use Node 8
                yarn build --environment=prod
            )
