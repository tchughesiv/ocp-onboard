Building or Deploying an Existing Container
===========================================

OpenShift can deploy existing code in a number of ways. Pre-built containers
stored in a Docker registry, such as `Docker Hub <https://hub.docker.com/>`_
can be downloaded and deployed directly to OpenShift. OpenShift can also
build images from source code in a `git <https://git-scm.com/>`_ repository,
regardless of whether or not a Dockerfile is present.

.. _deploy_dockerhub:

Deploying from Docker Hub
-------------------------

The simplest way to get an existing image into OpenShift is to retrieve
the image from Docker Hub. OpenShift will automatically create a new
:term:`image stream` and map to the image in Docker Hub.

Example
~~~~~~~

Create a new application in the current project, specifying the name of the
image in Docker Hub::

  $ oc new-app jdob/python-web

That's it. OpenShift will take care of retrieving the image and setting up
all of the necessary resources to deploy pods for the application.

.. include:: includes/route-warning.txt

.. _build_dockerfile:

Building from a Dockerfile in OpenShift
---------------------------------------

When creating a new application, OpenShift can be passed the location of a
git repository to automatically trigger a new image build. The type of build
performed will depend on what is found in the repository.

If a Dockerfile is present, OpenShift will perform the following steps:

* A :term:`build configuration` is created to correspond to building an image
  from the repository's Dockerfile. The location of the repository, as well
  as the settings identifying the build as a Docker build
  (``Strategy: Docker``), will be present.
* An :term:`image stream` is created to reference images built by the
  configuration, which is later used as a trigger for new deployments. In
  practice, when a new build is performed, the application will be redeployed
  with the new image.
* The first build of the created configuration is started.
* The remainder of the necessary components, as described in :doc:`anatomy`
  are created, including the replication controller and service.

Example
~~~~~~~

The standard command for creating an application is used, but instead of
referencing a specific image to deploy, the URL to the git repository is
provided::

  $ oc new-app https://github.com/jdob-openshift/python-web

Of particular interest in this example is the created build configuration.
The list of build configurations can be found using the commands in the
:ref:`resource_query_commands` section and specifying the ``buildconfig``
(or ``bc`` for short) type::

  $ oc get buildconfig
  NAME               TYPE      FROM      LATEST
  python-web         Docker    Git       1

Specific details about the build configuration are retrieved using the
``describe`` command and the name of the configuration itself:

.. code-block:: none
   :emphasize-lines: 9,10,11

   $ oc describe bc python-web
   Name: python-web
   Namespace: dockerfile-build
   Created: 49 minutes ago
   Labels: app=python-web
   Annotations: openshift.io/generated-by=OpenShiftNewApp
   Latest Version: 1

   Strategy: Docker
   URL: https://github.com/jdob-openshift/python-web
   From Image: ImageStreamTag openshift/python:latest
   Output to: ImageStreamTag python-web:latest

   Build Run Policy: Serial
   Triggered by: Config, ImageChange
   Webhook GitHub:
     URL: https://localhost:8443/oapi/v1/namespaces/dockerfile-build/buildconfigs/python-web/webhooks/iqQhagYGyA4OrZ3jXZpa/github
   Webhook Generic:
     URL: https://localhost:8443/oapi/v1/namespaces/dockerfile-build/buildconfigs/python-web/webhooks/B1KFkXvJmi_5KycoFYkw/generic
     AllowEnv:	false

A few notes on the highlighted values above:

* The ``URL`` value corresponds to the git repository where the source is found.
* The strategy is set to Docker, indicating that the image should be built
  using a Dockerfile found in the repository. For comparison, see the
  :ref:`build_s2i` section.
* The ``From Image`` is the base image for the build and is derived directly
  from the Dockerfile.

It is also worth noting that webhook URLs are provided. These URLs can be
used to inform OpenShift of changes to the source code and trigger a new
build. Depending on the :term:`deployment configuration`, new images
built from this trigger will automatically be deployed (this is the default
behavior for the deployment configuration automatically created by this
process). GitHub, for example, supports adding this URL to a repository to
automatically trigger a new build when a commit is pushed.

.. include:: includes/route-warning.txt

.. _build_s2i:

Source to Image
---------------

OpenShift's Source-to-Image (S2I for short) functionality takes the ability to
build images from a repository one step further by removing the need for
an explicit Dockerfile.

Not all Docker images can be used as the basis for an S2I build. Builder
Images, as they are known, have minimal but specific requirements regarding
files that OpenShift will invoke during the build. More information on
creating new builder images can be found in the create_builder_image
section.

There are two ways to determine the base image that will be used in the build:

* Explicitly specifying the image when the application is created.
* If no base image is indicated, OpenShift will attempt to choose an
  appropriate base. For example, the presence of a ``requirements.txt`` file
  will cause OpenShift to attempt to use the latest Python builder image
  as the base.

Once the git repository (and optionally, an explicit base image) are specified,
OpenShift takes the following steps:

* A container is started using the builder image.
* The source code is downloaded from the specified repository and injected
  into the container.
* The builder container performs the appropriate steps depending on the base
  image. Each builder image includes scripts that perform the necessary steps
  to install applications of the supported type. For example, this may include
  installing Python packages via pip or copying HTML pages to the configured
  hosting directory.
* The builder container, now containing the installed source code, is committed
  to an image. The image's entrypoint is set based on the builder image (in
  most cases, this is defaulted to a standard but can be overridden using
  environment variables).

As with :ref:`build_dockerfile`, the other components of a service are
automatically created, a build is triggered, and a new deployment performed.

.. warning::

  It is important to understand how base images are configured. Since there
  is no user-defined Dockerfile in place, applications are restricted to
  using ports exposed by the builder image. Depending on the application,
  it may be useful to define a custom builder image with the
  appropriate ports exposed.

Example
~~~~~~~

The standard command for creating an application is used, but instead of
referencing a specific image to deploy, the URL to the git repository is
provided::

  $ oc new-app python:3.4~https://github.com/jdob-openshift/python-web-source

Note the snippet preceding the git repository URL. This is used to tell
OpenShift the base image to build from and is indicated by adding the
image and tag before the repository URL, separated by a ``~``.

The build configuration for an S2I will differ slightly from one built from
a Dockerfile:

.. code-block:: none
   :emphasize-lines: 9,10,11

   $ oc describe bc python-web-source
   Name: python-web-source
   Namespace: source-build
   Created: 6 minutes ago
   Labels: app=python-web-source
   Annotations: openshift.io/generated-by=OpenShiftNewApp
   Latest Version: 5

   Strategy: Source
   URL: https://github.com/jdob-openshift/python-web-source
   From Image: ImageStreamTag openshift/python:3.4
   Output to: ImageStreamTag python-web-source:latest

   Build Run Policy: Serial
   Triggered by: Config, ImageChange
   Webhook GitHub:
     URL: https://localhost:8443/oapi/v1/namespaces/web2/buildconfigs/python-web-source/webhooks/AipYevj9pknT6SDWJNR0/github
   Webhook Generic:
     URL: https://localhost:8443/oapi/v1/namespaces/web2/buildconfigs/python-web-source/webhooks/R0U9P0QgPy3ncXMQphnC/generic
     AllowEnv: false

A few notes on the highlighted values above:

* The ``URL`` value corresponds to the git repository where the source is found.
* The strategy is set to Source, indicating that the image should be built
  against a builder image.
* The ``From Image`` is the builder image for the build. This value can be
  derived automatically by OpenShift or, in this example, explicitly set
  when the application is created.

As noted in :ref:`build_dockerfile`, the webhook URLs can be used to have
the source repository automatically trigger a new build when new commits
are made.

.. note::

  Unlike deploying an existing image, applications created through S2I
  are automatically configured with a route.

Next Steps
----------

Now that an application is deployed, see the :doc:`anatomy` section guide
for more information on the different resources that were created, or the
:doc:`basic-usage` guide for other ways to interact with the newly
deployed application. Information on enhancing container images to utilize
OpenShift's features can be found in the :doc:`integration` guide.
