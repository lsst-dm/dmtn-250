:tocdepth: 1

Abstract
========

The Rubin Science Platform is composed of a large number of separate services, some following external standards such as IVOA and some providing internal protocols.
The services available and the data behind them will vary by site and need to be discoverable by users.
This tech note lists known discovery needs for both services and schemas and proposes an implementation plan for meeting those needs.

In some cases, this discovery service has already been implemented.
In other cases, this design is tentative and is likely to change during implementation.
Each implementation section describes the current state for that discovery service.

VO services
===========

The IVOA has a set of specifications for service discovery.
Individual services can advertise their service capabilities using VOSI_.
A Registry_ can then incorporate those capability records along with additional metadata and present them in the form of VOResource_ entities.

.. _VOSI: https://www.ivoa.net/documents/VOSI/
.. _Registry: https://www.ivoa.net/documents/RegistryInterface/
.. _VOResource: https://www.ivoa.net/documents/REC/ReR/

The IVOA also defines a RegTAP_ protocol for searching the registry using the `Table Access Protocol`_.

.. _RegTAP: https://www.ivoa.net/documents/RegTAP/
.. _Table Access Protocol: https://www.ivoa.net/documents/TAP/

Each deployment of the Rubin Science Platform should provide an IVOA Registry advertising all IVOA services (and similar services for which it makes sense to write IVOA-compatible metadata) available in that deployment.
Some Science Platform deployments may be registered with the `Registry of Registries`_.
Most will not be, usually because they're for internal use only, but the local registry can still be used by local VO clients.

.. _Registry of Registries: https://www.ivoa.net/documents/Notes/RegistryOfRegistries/

.. _vo-implementation:

Implementation
--------------

We have a few VO services that provide capabilities endpoints, but have not yet implemented an IVOA registry service.
Here is a proposed design.

VOSI_ standardizes a capabilities endpoint for each IVOA service.
We want to avoid maintaining service capabilities in multiple locations, so the registry service should gather capabilities information from each running service and assemble it into a registry.

To do this, the registry service needs a list of capabilities URLs for all services running in that instance of the Science Platform, since the services deployed will vary by instance.
The canonical source of this information is the Phalanx_ configuration for that Phalanx environment.
We will use Helm to translate the list of enabled applications in Phalanx into a list of URLs injected into the configuration of the registry service via Helm values set by Argo CD, similar to how we're using Argo CD to inject configuration into the Telegraf service (see :sqr:`061`).
Additional configuration can be added by the Helm configuration of the registry service for each environment.

.. _Phalanx: https://phalanx.lsst.io/

The registry service itself will be a simple read-only FastAPI_ service, possibly merged with some or all of the other HTTP-based discovery services described in this tech note.

.. _FastAPI: https://fastapi.tiangolo.com/

This leaves the question of how to implement RegTAP_.
We want to handle RegTAP in much the same way that we handle :ref:`the TAP schema <tap-schema>`: a separate MySQL server with tables containing the required information that is used as a backend by the existing TAP service, so that we don't have to maintain a separate TAP server implementation.

The registry service knows all of the information needed for that MySQL table, and we don't want to have to maintain that information in multiple places.
Therefore, the registry service, after it assembles the XML document that it will serve from the capabilities endpoints of the services available in that environment, should replace the registry table in the MySQL service with a SQL version of the same data.
From there, it can be queried by the TAP service similar to the TAP schema table.

.. _tap-schema:

TAP schema
==========

The TAP service itself can be queried for its own schema by querying the ``TAP_SCHEMA`` schema.
This is specified in the TAP_ standard.
We therefore only need to populate the ``TAP_SCHEMA`` table and serve it through the TAP service.

.. _TAP: https://www.ivoa.net/documents/TAP/

Observation data will be served by a separate ObsTAP_ service that needs its own ``TAP_SCHEMA`` table containing only the schema supported by that TAP service.
See :dmtn:`236` for more information about the ObsTAP design.

.. _ObsTAP: https://www.ivoa.net/documents/ObsCore/

As discussed in :ref:`VO services <vo-implementation>`, we will also want to serve RegTAP_ data.
The schema for that data will also need to be part of ``TAP_SCHEMA``.

Implementation
--------------

Schemas for the Rubin Science Platform are managed using Felis_ from metadata stored in the `sdm_schemas repository`_.
As part of the build process of that repository, Felis generates MySQL tables holding the schema information and bundles them into a Docker image for a MySQL server that contains only the ``TAP_SCHEMA`` table.
This server is deployed in Phalanx_ as the tap-schema application, and the TAP server is configured to point to it.

.. _Felis: https://felis.lsst.io/
.. _sdm_schemas repository: https://github.com/lsst/sdm_schemas

Currently, that schema does not contain RegTAP_ schema, since we're not serving RegTAP data.
When we start doing so, we can incorporate that schema into the ``TAP_SCHEMA`` table generated from the `sdm_schemas repository`_.
In order to serve the RegTAP data itself out of the same MySQL service used for ``TAP_SCHEMA``, we may have to make additional modifications to the TAP service to add additional backend data sources.

How to provide ``TAP_SCHEMA`` for the ObsTAP service is not yet clear.
The simplest approach will be if the PostgreSQL backend for ObsTAP can itself provide a ``TAP_SCHEMA`` table as well.
Then, the ObsTAP service can use a single backend for all data.

If that's not possible, we will need to create a separate obstap-schema Phalanx service and build a separate set of MySQL images that contain the ObsCore schemas instead of the schemas for the regular TAP service.

DataLink metadata
=================

We use the IVOA DataLink_ protocol both to serve images corresponding to TAP query result rows and to link results to other related services such as a SODA_ cutout service.
See :dmtn:`238` for more details.

.. _DataLink: https://www.ivoa.net/documents/DataLink/
.. _SODA: https://www.ivoa.net/documents/SODA/

The second part of the use of DataLink requires sending metadata to the TAP service, so that it knows what DataLink records to return with result rows.
The DataLink service itself also needs metadata about the TAP schema in order to implement several microservices that provide convenient navigational links from result rows to related TAP queries.

.. _datalink-implementation:

Implementation
--------------

Currently, each release of sdm_schemas_ also generates, via GitHub Actions, two additional release artifacts: :file:`datalink-snippets.zip` and :file:`datalink-columns.zip`.
The TAP service downloads the former on restart and uses it to control which DataLink records are added to result tables.
The DataLink service downloads the latter on restart and parses it for information about the TAP schema.

.. _sdm_schemas: https://github.com/lsst/sdm_schemas

This works, but it requires manual configuration changes to both the TAP and DataLink services in Phalanx each time there is a new relevant sdm_schemas release.
Doing this via Helm chart configuration also makes it awkward to use different versions of the data in different environments.
If someone forgets to change the versions of the data downloaded by those services, they may also use metadata that is out of sync with the ``TAP_SCHEMA`` metadata provided by the tap-schema application.

Instead, sdm_schemas should build a Docker image of a small static file web server with this data included as part of its build process.
This static file web server would then be kept in sync with the MySQL server Docker image by deploying both via the tap-schema application with the same version numbers.
The TAP service and DataLink service should then query that service (possibly with a local cache) for the relevant metadata when it is needed.
This will also allow updating the metadata without having to remember to manually restart the services.

Butler
======

The Rubin Science Platform uses Butler_ to manage and manipulate data files and formats.
Butler currently is a library that makes direct SQL connections to a backing SQLite or PostgreSQL database, but eventually will support a client/server web protocol (see :dmtn:`176` and :dmtn:`242`).

The Butler client library will support either direct access to a local database or acting as a client to a Butler web service.
Which protocol to use for a given operation is determined by a configuration file that maps Butler repository names to access information.
Here is one of those configuration files as an example:

.. code-block:: yaml

   dp01: "s3://butler-us-central1-dp01"
   dp02-test: "s3://butler-us-central1-dp02-user"
   dp02: "s3://butler-us-central1-dp02-user"

Additional configuration information for a given Butler repository is provided by a :file:`butler.yaml` file at the root of the repository.
For example, here is a portion configuration file for the ``dp02`` Butler repository (lines not relevant to this discussion have been removed; the actual file is more complex than this):

.. code-block:: yaml

   datastore:
     datastores:
       - datastore:
           name: FileDatastore@s3://butler-us-central1-panda-dev/dc2
           root: s3://butler-us-central1-panda-dev/dc2
       - datastore:
           name: FileDatastore@s3://butler-us-central1-dp02-user
           root: s3://butler-us-central1-dp02-user/
           records:
             table: user_datastore_records
   registry:
     db: postgresql://postgres@10.163.0.3/idfdp02

Client/server Butler will have a similar mapping.

For each deployment of the Rubin Science Platform, we want to provide users with a Butler configuration file listing all of the Butler repository names that are valid for that deployment and their correct locations.
The same name may map to different locations in different deployments; for example, the ``dp02`` name may map to a Butler server local to that deployment of the Science Platform, whose URL will therefore vary.
Some deployments may have Butler repository names not found in any other deployment.

This configuration is used in the Notebook Aspect by arbitrary user notebooks, and by Science Platform services that require a Butler.
This is currently the SODA cutout service and the DataLink service, but will probably include other services in the future.

Implementation
--------------

Current state
^^^^^^^^^^^^^

Currently, these Butler configuration files are stored in a Google Cloud Storage bucket.
The ``s3`` URL to the file in that bucket is injected to user notebooks and Science Platform services via an environment variable, which is used by Butler to retrieve its configuration when the library is initialized.

This implementation has several problems:

- Every environment has to have a separate configuration file.
  These configuration files are currently manually maintained without a structured deployment process, linting, testing, etc.
  Changes to add new Butler repositories have to be done across all relevant environments.

- Some information that changes by environment is included in the :file:`butler.yaml` file at the root of the repository rather than in the top-level configuration file.
  For example, as seen above, the database for the registry points to a PostgreSQL server at a specific IP address, but that address is specific to one environment and will be different in other environments, even though the underlying database is the same.
  (For example, other environments may use a locally-hosted Google Cloud SQL Proxy server.)

- Generating and managing these configuration files is decoupled from Phalanx configuration even though Phalanx is a primary consumer of these files.
  This means there is more configuration that has to be done outside of Phalanx in order for the Science Platform to be fully functional, including URLs that have to be injected via per-environment :file:`values-{env}.yaml` files.

Proposed replacement
^^^^^^^^^^^^^^^^^^^^

A better Butler discovery process would allow easy per-environment configuration of known Butler repositories in one place, using Helm configuration via :file:`values.yaml` and :file:`values-{env}.yaml` and the normal Phalanx separation of general configuration and per-environment configuration.
The resulting Butler configuration would be served by a service local to that deployment of the Science Platform, with a known URL that can be injected into every service that needs it.
The Butler client used by that service would then load the Butler configuration from that URL and get the correct per-environment settings.

Provided that Butler can retrieve its configuration file via HTTP, doing this for the top-level configuration is straightforward.
However, currently it needs to be done for the next-level :file:`butler.yaml` file as well so that properties specific to the environment, such as the hostname or IP address of the PostgreSQL server, can be similarly customized.
It's not yet clear how best to do this; two obvious approaches are moving that information to the top-level configuration file, or templating the :file:`butler.yaml` configuration file by adding environment-specific information to a template provided by the underlying data store.

The static file server provided by the sdm_schemas repository in the proposed implementation for :ref:`DataLink metadata <datalink-implementation>` could also serve this Butler configuration.
The configuration file is static once constructed from the Phalanx configuration, and could be injected into the file space of the static file server by the Kubernetes deployment via a ``ConfigMap``.

EFD
===

The Engineering Facilities Database is used internal to Rubin Observatory for information used by project staff, such as telemetry from sensors and devices on the summit and performance metrics for the processing pipeline.
Unlike most other Science Platform use cases, it's often necessary for a user running a notebook in the Notebook Aspect of one Science Platform to connect to the EFD service provided by a different instance of the Science Platform.

When a user wants to access the EFD, by default they should be directed to the local instance.
However, if they request a specific instance, they should be directed to that instance.

This includes a requirement for authentication to the remote instance.
Currently, authentication is done by username and password, using a shared read-only account.

.. _efd-implementation:

Implementation
--------------

Currently, the Segwarides_ service running in Roundtable_ provides both discovery and authentication credentials for all EFD instances.
A client, at any Science Platform (or outside of any of them), tells Segwarides what EFD they want to connect to, and Segwarides returns the connection and authentication information for that EFD instance.
Normally, this is done via lsst-efd-client_.

.. _Segwarides: https://github.com/lsst-sqre/segwarides
.. _Roundtable: https://roundtable.lsst.io/
.. _lsst-efd-client: https://efd-client.lsst.io/

This approach has two problems.
First, it requires running a global Segwarides service, which in turn creates cross-domain authentication issues that we are currently ignoring.
Second, this architecture does not support the desired property of directing the user to the local instance by default, since it doesn't know which instance is local.

A better implementation would be to move the function of Segwarides to a service that runs in each Science Platform from which users may access an EFD.
That service would provide endpoints to retrieve the name and connection information for the local EFD, a list of known EFDs, or the connection information for another EFD by name.
This service would be protected by the authentication of that instance of the Science Platform (see :dmtn:`234`).

For the time being, we will continue to use username and password authentication, but this approach opens the possibility of using the user's existing authentication token for access to the local EFD.
Eventually, rather than returning a static username and password, it could also return a static authentication token for a remote EFD.

The drawback of this approach is that it requires duplicating the connection and authentication information for every EFD in each environment that runs an EFD discovery service.
That however can be handled automatically by Phalanx's Helm chart and secret automation, so while the operational data is duplicated, it will still be maintained by humans in only one place.

This EFD discovery service can be combined with the sdm_schemas service used for DataLink metadata and Butler discovery.
The data about known EFDs and their connection and authentication information can be injected into the Kubernetes deployment and parsed by a small amount of code in the sdm_schemas web service.
(This will technically make it not entirely a static file web service, but not add much meaningful complexity.)

Helper library
==============

Users could use the various discovery facilities directly, but to simplify writing notebooks in the Notebook Aspect, we provide helper functions to query Science Platform services.
For VO services, users could query the registry directly with PyVO_, but this is a somewhat complex interface that we want to simplify.

.. _PyVO: https://pyvo.readthedocs.io/en/latest/

These helper functions should be simple wrappers around queries to the above discovery services.
Butler is a special case since the Butler library itself will directly load the top-level configuration and can be configured (via an environment variable) with its location, and thus doesn't need a separate helper library.

Implementation
--------------

The current implementation of this helper library lives in two packages: lsst.rsp_ and lsst_efd_client_.
The former currently provides ways to access the TAP service (``lsst.rsp.get_tap_service`` and ``lsst.rsp.retrieve_query``).
The latter wraps the current EFD discovery service and provides methods to perform various types of EFD queries.

.. _lsst.rsp: https://github.com/lsst-sqre/lsst-rsp
.. _lsst_efd_client: https://efd-client.lsst.io/

We expect to want to add support for additional services to the lsst.rsp package, similar to the support for the TAP service.
The exact details may depend on the service; there are a few options for what may be most convenient for the user:

#. A mechanism to obtain the URL of the service, leaving all other details including authentication to the user.
   This gives the user the most flexibility, at the cost of only wrapping the discovery step and leaving authentication and all other details of service access to the user.
   This is the approach taken by the existing ``lsst.rsp.get_tap_service``.

#. A way to initiate a third-party client with the details required to access a service.
   For example, the helper function may return a PyVO_ client object with the service URL and authentication credentials already configured, or initiate the standard operation in a third-party client and return the results.
   This is the approach taken by the existing ``lsst.rsp.retrieve_query``.

#. Provide a full-fledged client for the service, which manages the connection and authentication and provides methods to perform all supported operations for that service.
   This is the approach taken by lsst_efd_client_.

There is no one solution that makes sense for every service.
Options 2 or 3 will make sense for some services but not others, depending on the details of that service, the availability of third-party libraries, and the anticipated use cases of users.

However, we can (and should) support option 1 for every service: return the URL of the named service in the local Science Platform, if it exists.
We can also support asking for a list of all services known to be running on the local instance of the Science Platform, where the result is a simple list of parameters that could be passed into the function asking for the URL.
This can wrap calls to the IVOA registry for VO services and other calls or sources of knowledge for non-VO services that don't make sense to register with the IVOA registry.

Rather than add a new function like ``lsst.rsp.get_tap_service`` for each individual service, this should be provided more generically, such as a ``get_service_url`` function that takes one of an enumerated list of services and returns the URL for that service if it is available.
