#################################################
Discovery services for the Rubin Science Platform
#################################################

.. abstract::

   The Rubin Science Platform is composed of a large number of separate services, some following external standards such as IVOA and some providing internal protocols.
   The services available and the data behind them will vary by site and need to be discoverable by users.
   This tech note lists known service discovery and schema discovery needs and proposes an implementation plan for meeting those needs.

In some cases, this discovery service has already been implemented.
In other cases, this design is tentative and is likely to change during implementation.
Each implementation section describes the current state for that discovery service.

Use cases
=========

.. _use-case-internal:

Internal service discovery
--------------------------

Applications running on an instance of the Rubin Science Platform often need to know the URLs of other services running on the same instance.

- datalinker_ needs the URL of the SODA cutout service to add to service descriptors included in the DataLink_ record returned by the ``{links}`` endpoint.
- The service that generates the HiPS list (see :ref:`use-case-hips-list`) needs to know the base URL of the HiPS service to retrieve and assemble the :file:`properties` files.
- Ghostwriter_ needs the base URL for a user's Nublado lab, the URL to the JupyterHub API to spawn a new lab, and the URL to the TAP service to construct the URL to a TAP query.
- Kafka clients that manage Avro schemas need the URL of the Confluent Schema Registry.
- mobu_ needs the URL to Gafaelfawr to obtain tokens, the JupyterHub API to spawn a new lab, the URL to the TAP service to make TAP queries directly, and the URL to the SIAv2 service to make SIAv2 queries directly.
- Noteburst_ needs the URL to the JupyterHub API to spawn a new lab, the URL to Gafaelfawr to obtain tokens, and the URL to the Nublado controller to get the list of available images.
- Nublado_ needs the URL to Gafaelfawr to log users out and the URL to the Nublado controller to manage user labs.
- The `Qserv Kafka bridge`_ needs the URL to Gafaelfawr to get user quota information and the base URL for datalinker to rewrite TAP query output columns to point to the correct datalinker service.
- Squareone_ uses the base URL to construct a variety of URLs for various internal services, including Gafaelfawr, the Portal, and Nublado.
  In the future, Squareone will use service discovery to determine what services are available in a given environment, and will customize its pages based on that information.
- All TAP servers (``tap``, ``ssotap``, ``livetap``, ``consdbtap``) need the URL to Gafaelfawr's token endpoint to get user information.
- `Times Square`_ needs the URL to Noteburst to run notebooks and the URL to its UI in Squareone to construct links for GitHub PRs.
- Unfurlbot_ needs the URL to the JIRA data proxy.
- vo-cutouts_ (and any other service using the `Safir UWS pattern`_) needs the URL to Wobbly for the UWS database API.

.. _datalinker: https://github.com/lsst-sqre/datalinker
.. _DataLink: https://www.ivoa.net/documents/DataLink/20231215/REC-DataLink-1.1.html
.. _Ghostwriter: https://ghostwriter.lsst.io/
.. _mobu: https://mobu.lsst.io/
.. _Noteburst: https://noteburst.lsst.io/
.. _Nublado: https://nublado.lsst.io/
.. _Qserv Kafka bridge: https://github.com/lsst-sqre/qserv-kafka/
.. _Squareone: https://squareone.lsst.io/
.. _Times Square: https://times-square.lsst.io/
.. _Unfurlbot: https://github.com/lsst-sqre/unfurlbot
.. _vo-cutouts: https://github.com/lsst-sqre/vo-cutouts
.. _Safir UWS pattern: https://safir.lsst.io/user-guide/uws/index.html

As a special case of this type of internal service discovery, the Butler client must be configured with a YAML file that controls many aspects of its behavior.
These YAML files are provided via HTTP by the Butler server and are specific to particular datasets.
Applications that use the Butler client must be configured with a mapping of dataset names to URLs to the corresponding Butler client configuration file.

The Portal needs to discover the locations of various other services, including HiPS and TAP servers, but it prefers to use IVOA-standard protocols.
This use case is discussed at :ref:`use-case-vo`.

One final special case is mobu_.
To determine which test notebooks to run, mobu needs to know the list of services deployed in a given instance of the Rubin Science Platform.
This is a form of service discovery and should be provided by the service that tracks all deployed components and their URLs.

.. _use-case-vo:

VO service discovery
--------------------

The IVOA has a set of specifications for service discovery.
Individual services can advertise their service capabilities via the VOSI_ capabilities endpoint.
A Registry_ can then incorporate those capability records along with additional metadata and present them in the form of VOResource_ entities.

.. _VOSI: https://www.ivoa.net/documents/VOSI/
.. _Registry: https://www.ivoa.net/documents/RegistryInterface/
.. _VOResource: https://www.ivoa.net/documents/VOResource/20250416/REC-VOResource-1.2.html

There are two general types of VO registries: a *searchable registry*, which supports general searches for datasets and services of interest; and a *publishing registry*, which publishes VOResource_ records with very limited list and retrieval APIs.
The latter is built on the OAI-PNH_ protocol.

.. _OAI-PNH: https://www.openarchives.org/OAI/openarchivesprotocol.html

Our immediate need for the Rubin Science Platform is two-fold:

#. VO-first services such as the Portal Aspect should ideally be able to use standard VO protocols to locate all necessary services for a given dataset within the same instance of the Rubin Science Platform.
   For our present purposes, hard-coding the identifiers of the available datasets in a given RSP deployment into the configuration is acceptable, but it should then be possible to find the various services given those identifiers and the location of the publishing registry.

#. Rubin Observatory should publish its astronomer-facing services and datasets to the astronomy community at large with appropriate metadata, including the authentication and data rights requirements.
   This data can then be harvested by non-Rubin IVOA services that provide searchable registries (through RegTAP_, for example) and `Registry of Registries`_ services.
   Providing this data requires understanding how to represent Rubin's authentication scheme in a VO-compatible way, which may in turn require some changes to the headers returned by services or by Gafaelfawr authentication challenges.

.. _RegTAP: https://www.ivoa.net/documents/RegTAP/
.. _Registry of Registries: https://www.ivoa.net/documents/Notes/RegistryOfRegistries/

We do not currently believe there is a need for a local searchable registry for each instance of the Rubin Science Platform.

The clients of interest for VO-based service discovery are the Portal Aspect (and any other future applications that are not specific to the Rubin Science Platform and prefer VO protocols), and third-party VO protocol clients such as TOPCAT.
It may also be useful for notebooks and programs written with PyVO_, possibly run within the Notebook Aspect, although see :ref:`use-case-helpers` for other approaches to that use case.

.. _TOPCAT: https://www.star.bris.ac.uk/~mbt/topcat/
.. _PyVO: https://pyvo.readthedocs.io/en/latest/index.html

.. _use-case-hips-list:

HiPS list generation
--------------------

HiPS data for a given data release is stored in Google Cloud Storage buckets.
Currently, this data is served by a trivial proxy named crawlspace_ that retrieves the data from GCS using private credentials and returns it to the user.
Eventually we hope to add authentication support to the native GCS mechanism for serving files via HTTPS directly and do away with this proxy.

.. _crawlspace: https://github.com/lsst-sqre/crawlspace

Because we serve the same HiPS data from multiple instances of the Rubin Science Platform, the GCS bucket does not contain the HiPS :file:`list` file, which is used by HiPS clients to discover the available HiPS trees.
Only the :file:`properties` files for the individual trees are present in the bucket.
We therefore need a service that assembles those :file:`properties` files into a HiPS :file:`list` file for client consumption, with the correct URLs inserted for the given environment.

.. _use-case-nublado:

Nublado extensions
------------------

The JupyterLab extensions added by Nublado (as opposed to helper libraries intended for the user, discussed in :ref:`use-case-helpers`) need to know several URLs to talk to other services:

- The ``displayversion`` extension puts an identifier (currently the hostname) of the Science Platform in the bottom bar of the JupyterLab UI and needs that information in client-side JavaScript.
- The ``savequit`` extension needs to know the logout URL in client-side JavaScript.
- The ``query`` extension needs the URLs to the times-square and TAP APIs in the server-side Python code.
  It currently only supports one TAP server.
- The ``ghostwriter`` server-side code needs the URL to the Ghostwriter_ API.
  It also needs to know the base URL for JupyterHub, although this is not precisely a service discovery case since this is another component of the same service.
- The ``firefly`` extension requires the URL to the Portal.

Nublado tutorials
^^^^^^^^^^^^^^^^^

The ``tutorials`` JupyterLab extension needs to know the structure of the tutorials GitHub repository so that it can assemble a menu for the user.
This is not directly related to Phalanx service discovery (the source information is at GitHub), but the way the information is currently being assembled is awkward and requires updating a directory on a shared NFS mount.

Since Repertoire provides a service that can cache and provide this information on demand, it may make sense for it to provide an endpoint that can be used by all user labs, avoiding the need to store the data in a shared file system or assemble it for each lab separately.

.. _use-case-helpers:

Python helper library
---------------------

The user notebook pods created by Nublado include a Python library, lsst-rsp_, that provides helper functions that aid in contacting Science Platform services.
These helper functions hide the details of authentication and service discovery for the user and should promote good style for user notebook code, such as explicitly requesting a specific data release.

.. _lsst-rsp: https://github.com/lsst-sqre/lsst-rsp

We expect to support all Rubin Science Platform services directly in this helper library, but the mechanism will vary by service.
There are four major cases:

#. A mechanism to obtain the URL of the service for a given dataset, leaving all other details including authentication to the user.
   This gives the user the most flexibility, at the cost of only wrapping the discovery step and leaving authentication and all other details of service access to the user.

#. A way to obtain a generic Python HTTP client that is preconfigured with the user's authentication credentials and the base URL of the service for a given dataset, but which does not otherwise wrap access to the service.
   This is less flexible since we are choosing the HTTP client implementation for the user, but it hides managing the authentication token, which we expect some users to struggle with.

#. A way to initialize and return a third-party client with the details required to access a service.
   For example, the helper function may return a PyVO_ client object with the service URL and authentication credentials already configured, or initiate the standard operation in a third-party client and return the results.

#. Provide a full-fledged client for the service, which manages the connection and authentication and provides methods to perform all supported operations for that service.

There is no one solution that makes sense for every service.
Options 2 or 3 will make sense for some services but not others, depending on the details of that service, the availability of third-party libraries, and the anticipated use cases of users.

In all cases, these helpers should use an underlying service discovery service and library to determine if a given combination of dataset is available in this instance of the Rubin Science Platform and, if so, return one of the four possible objects listed above.
This hides more complexity from the user, and provides us with more implementation flexibility, than if the user used the discovery service and its library directly.
For VO services, users could instead query the registry directly with PyVO_, but this is a somewhat complex interface that we want to simplify and all available services may not be registered with IVOA registries or representable in that service discovery system

.. _use-case-sasquatch:

Sasquatch
---------

Sasquatch provides Kafka and InfluxDB databases for several purposes.

Service discovery of Kafka, which is also the mechanism used for writing to InfluxDB, should be done via strimzi-access-operator.
The secret created by strimzi-access-operator covers all of the required information except for the Confluent Schema Registry, for those applications that use Avro schemas.
The address of the Confluent Schema Registry can be handled via :ref:`use-case-internal`.

Query access to the InfluxDB databases from, for example, notebooks will require service discovery of the available databases and their paths and connection information.
InfluxDB databases require authentication.
Currently, we use username/password authentication, so a client that wants to query the InfluxDB database needs some mechanism to acquire that password.

Service discovery for Sasquatch InfluxDB databases should therefore return the following information:

- InfluxDB URL (some clients prefer this as hostname, port, and path, so provide it in both forms)
- Username
- Password

Because a password is included, this service discovery API, unlike the API for internal service discovery more generally, must be authenticated.
Clients that also need the Confluent Schema Registry URL should discover that through the regular internal service discovery API.

The advertised InfluxDB databases should be filtered by the user's scopes.
For example, the application metrics InfluxDB database should only be available to environment administrators, not to general users of the environment.

EFD
^^^

The Engineering Facilities Database is used internal to Rubin Observatory for information used by project staff, such as telemetry from sensors and devices on the summit and performance metrics for the processing pipeline.
Unlike most other Science Platform use cases, we currently support accessing an InfluxDB EFD database hosted in one Phalanx instance from a different Phalanx instance (the USDF EFD database from the Summit, for instance).

When a user wants to access the EFD, by default they should be directed to the local instance.
However, if they request a specific instance and that instance is available from their local instance, they should be directed to that instance.

The InfluxDB credentials may therefore be for a service running at a separate Science Platform instance.
This use case is specific to the EFD.
The user should be able to authenticate with their local credentials and obtain the authentication credentials to use for the remote database (generally a remote EFD).

.. note::

   We should reconsider whether the remote access case is truly necessary, since supporting it increases the complexity of the system and requires syncing passwords between environments.

.. _use-case-tap-schema:

TAP schemas and associated metadata
-----------------------------------

Each TAP service has an associated schema, which itself is a database table named ``TAP_SCHEMA`` that can (and regularly is) queried by users and TAP clients.
This table is read-only for the TAP service and needs to be populated with a correct TAP schema matching the underlying database queried by that TAP server.
For Rubin, these schemas are managed in the sdm_schemas_ repository and published as releases from that repository.

.. _sdm_schemas: https://github.com/lsst/sdm_schemas

We often want to use different versions of the published schema in different Science Platform environments so that we can, for example, test new versions of the schema in non-production environments before deploying them.
We therefore need a mechanism to populate the ``TAP_SCHEMA`` database used by each TAP server with the schema from a release tag or branch of the sdm_schemas_ repository.
Changes to the contents of that repository should be atomic with respect to user queries: either they get the old schema or the new schema, but not an inconsistent intermediate result.

Additional TAP metadata
^^^^^^^^^^^^^^^^^^^^^^^

In addition, there are two other collections of data associated with the TAP schema:

- If certain database columns are included in a TAP result, that result should contain additional service descriptors that point the user to other services that may be used in combination with that data.
  This addition is done by the TAP server when constructing the result footer, but the metadata for what service descriptors to include is maintained in sdm_schemas_.
  That metadata needs to be available to the TAP server and match the current TAP schema.

- Some services that wrap TAP queries need to know what sets of columns to include in their results and how to order those columns.
  This metadata is derived from the table schemas maintained in sdm_schemas_.
  That derived metadata needs to be made available to those services.

.. _use-case-docs:

Documentation
-------------

We embed URLs to services in several generated documentation sites:

- `Phalanx <https://phalanx.lsst.io/>`_ (in various places), derived from the Phalanx configuration itself.
- `Sasquatch <https://sasquatch.lsst.io/environments.html>`__, maintained by hand.
- `rsp.lsst.io <https://rsp.lsst.io/>`__, manually maintained in JSON format.

Documentation sites need statically-generated information for multiple environments, so cannot easily use the service discovery API directly.
Ideally, however, the links in documentation should be built on the same underlying data and automatically propagate to all documentation sites when the underlying data changes.

rsp.lsst.io currently uses the following information for each environment:

- Name, short title, and long title
- Parent domain
- Squareone URL
- Portal URL
- Nublado URL
- API URL
- TAP URL
- Gafaelfawr token UI URL
- Times Square URL
- Phalanx documentation URL

Implementation proposal
=======================

Most service and data discovery services will be handled by a new FastAPI application, tenatively named Repertoire.
Repertoire will be deployed, as a Phalanx application, in each instance of the Rubin Science Platform that requires service and data discovery.
Since this includes mobu, it will be part of the ``infrastructure`` application group and normally deployed on every Science Platform instance.

The list of services enabled for that instance, and any other required metadata about the host or URL layout of that instance, will be injected into Repertoire by Phalanx via Argo CD.
Using that data, as well as Phalanx configuration and secrets, Repertoire will then provide the various service and data discovery APIs as described below.
Repertoire will not do any data discovery or dynamic analysis of the environment; all data that it provides must come from its Phalanx confiugration and built-in rules to derive service URLs from Phalanx configuration information.

Repertoire will also provide a client library, available from PyPI as ``rubin-repertoire``, that can be used to easily query Repertoire for service and data discovery implementation.

At least for now, Repertoire will not provide availability information.
We have other plans related to availability and will initially develop that as a separate system with its own tech note.
Once that work is complete, we will evaluate whether it makes sense to integrate some availability information into service discovery.

.. _implementation-internal:

Internal service discovery
--------------------------

Currently, Phalanx injects a list of services deployed in a given instance of the Rubin Science Platform into mobu.
Instead, Phalanx should use a similar technique to inject that list into the local instance of the new Repertoire service.

Repertoire will then construct a data model with the following components:

- The top-level name of this instance of the Rubin Science Platform, used for identity in browser UIs.
- The global logout URL for this instance of the Rubin Science Platform.
- A list of datasets available to this instance of the Rubin Science Platform.
  This should include a mapping to their IVOA IDs, as used by VO services.
- For each service that queries or otherwise interacts with a dataset, a map of service name to datasets to service API URLs.
  For the Butler server, as a special case, this will be the URL from which the Butler client configuration for that dataset can be retrieved.
  Services that share the same URL for all datasets will have a mapping for every known dataset.
  By default, the key should match the name of the Phalanx application.
  In cases where there is more than one service provided by the same Phalanx application, append ``-`` and an additional qualifier.
- For services that will never have separate per-dataset URLs, such as the Portal or Nublado, a mapping from service names to base URLs.
- A simple list of all deployed services (Phalanx application names) for the use of mobu and other services that need to know a complete list of what is deployed in that instance of the Rubin Science Platform, even if it does not have an API URL.

.. note::

   It's not clear whether to name specific services in the service discovery API model or to model the services as a mapping of generic service names to datasets to URLs.
   The advantage of the former is that we can attach documentation for how to use the result for a specific service and limit the discovered services to ones with known semantics, thus discouraging dumping arbitrary mappings into service discovery and creating a backwards-compatibility burden.
   The drawback of the former is that a revision of the Repertoire service and client library would be required to add a new service, and the client would have to be updated in all callers that needed to know about the new service, including user notebook environments.

Finally, the URL to the service discovery API should be injected into every Phalanx application that may need service discovery, replacing the current injection of ``global.baseUrl``.
Applications are encouraged to use the service discovery client library instead of making direct calls to the service discovery endpoint and parsing the results.
The service discovery client library will, based on integration experience, provide a simple API to retrieve and cache discovery information and return an appropriate URL or pre-configured client for another Phalanx service.

.. note::

   The exact API of the client library is left unspecified so that it can be formed through implementation experience.
   When it has stabilized, this tech note will be updated with a link to the API documentation.

Service discovery should not be used for configuration specific to one Phalanx application, such as the location of an application-specific PostgreSQL database or the URL of the Qserv used by an instance of the Qserv Kafka bridge.
It should only be used for locating other services within the same instance of the Rubin Science Platform.
Similarly, service discovery should not be used for secrets; for those, use `Phalanx secrets management <https://phalanx.lsst.io/developers/helm-chart/define-secrets.html>`__.
(The EFD is a special exception; see :ref:`use-case-sasquatch` and :ref:`implementation-sasquatch`.)

Path prefixes for services
--------------------------

Most Phalanx services written by SQuaRE support configuring the path prefix at which they listen.
That configuration is exposed via :file:`values.yaml` in their Phalanx configuration.

In order for service discovery to support reconfiguration of the default paths, Repertoire either has to use dynamic service discovery (see :ref:`dynamic`) and use that to update its expectations of service paths, or it needs to consume the Phalanx configuration for every service.
The latter is very awkward given the structure of Phalanx and the restrictions of Argo CD and would require a lot of complexity that seems likely to cause longer-term issues.

.. note::

   For the time being, service discovery will ignore custom path prefix configurations and therefore will be incorrect for Phalanx environments that use that configuration.
   We will revisit this when adding dynamic service discovery.

Kafka
-----

Continue to use the secrets provided by ``strimzi-access-operator`` for service discovery of the Kafka bootstrap servers rather than using Repertoire.

For the time being, applications that manage Avro schemas should continue to hard-code the internal URL of the Confluent Schema Registry into their configuration rather than using service discovery.
This may be switched to service discovery once authentication has been implemented.

.. _implementation-vo:

VO service discovery
--------------------

The basis of VO service discovery is the VOResource_ record and its associated extensions, which in turn is partly based on the VOSI_ capabilities record.
The first step of implementing VO service discovery will therefore be to ensure that all relevant services provide the VOSI capability endpoint, even if they are normally discovered via other means (such as DataLink_ service descriptors).

Then, Repertoire, knowing the API URLs of the services and the services deployed in a given instance of the Rubin Science Platform (see :ref:`implementation-internal`), along with additional configured metadata to flesh out the VOResource records, can construct the records for a VO publishing registry for that instance of the Rubin Science Platform.

Repertoire will therefore also provide an OAI-PNH service (on a different URL than internal service discovery).
For astronomer-facing installations of the Rubin Science Platform, this service will be public so that it can be queried by VO searchable registries and the Registry of Registries.
This API may require authentication on other instances of the Rubin Science Platform where VO services are not intended to be available to the astronomy community.

Only VO services that should be advertised to the astronomy community will be included in the OAI-PNH API.
This will therefore be a subset of the list of services and datasets contained in the internal service registry, but augmented with the additional metadata required by VOResource.

There are several Python OAI-PNH implementations that may be useful as a starting point:

- `IVOA publishing registry code <https://github.com/ivoa/publishing-registry>`__
- `NOIRLab VO registry <https://gitlab.com/nsf-noirlab/csdc/vo-services/noirlab-vo-registry>`__
- `pyoai <https://pypi.org/project/pyoai/>`__

Given the tight integration with internal service discovery and the desire to avoid manual metadata collection processes in favor of dynamically generating the OAI-PNH information based on internal service discovery, it is unlikely that any of these implementations can be used as-is, but they may be useful as a basis for the Repertoire implementation.

.. _implementation-hips:

HiPS list generation
--------------------

The top-level HiPS list is a collection of all of the HiPS :file:`properties` file served by that instance of the Rubin Science Platform, edited to insert the correct URLs.
This is essentially a service (or data) discovery service.

For each environment with a HiPS service, Repertoire will be configured with a list of datasets and, for each, the base URL of the HiPS trees and a list of HiPS trees.
As part of its internal service discovery implementation, it will know the base URL to the HiPS service for that RSP environment.
It will retrieve the :file:`properties` files for each tree and assemble them into one HiPS list file per dataset.

We will hopefully be able to retire the legacy ``/api/hips/list`` API as part of the transition to Repertoire.

.. _implementation-nublado:

Nublado extensions
------------------

Any environment variables set for the use of Nublado extensions will also become part of the user's environment and therefore implicitly become a user-facing API that imposes an ongoing, long-term maintenance burden.
We should therefore minimize the use of environment variables whenever possible, preferring approaches that do not leak into the user environment, unless the environment variable is intended to be a user-facing API.

Both the server side of the Nublado extensions and the user-facing Python helper library (see :ref:`implementation-helpers`) will need to know how to query service discovery.
This makes that one URL part of the user-facing API and therefore reasonable to provide in an environment variable.
That environment variable will be ``RUBIN_SERVICE_DISCOVERY_URL`` and be provided by the Nublado controller, which in turn will be configured with the service discovery URL of its Science Platform instance via its Helm chart.

Firefly
^^^^^^^

The Firefly extension supports configuring the URL to the Portal through the JupyterLab configuration.
This avoids making that configuration part of the user-facing API, so we should use this in preference to setting an environment variable.
Lab startup should use ``RUBIN_SERVICE_DISCOVERY_URL`` to discovery the base URL for the Portal and then set the appropriate JupyterLab configuration variable (``Firefly.url``).

Previously, we set the ``FIREFLY_URL`` environment variable.
We should drop that setting if possible, since environment variables will become implicit user-facing APIs accidentally.
If we cannot do so for unforseen backwards-compatibility reasons, ``FIREFLY_URL`` should be set during lab startup based on the results of service discovery.

Passing information to JavaScript
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since there is no need for the non-EFD service discovery endpoint to require authentication, JavaScript extensions could request and parse service discovery information directly.
Currently, this only applies to the ``savequit`` extension, which needs the logout URL.
(The Firefly extension already has its own internal mechanism for service discovery of the Portal.)
This would allow the server side of the Nublado extensions to drop its handlers for providing service discovery information.

The other place that a service discovery URL is used, the ``displayversion`` extension, should be replaced with Python code that calculates the version information (including the base hostname of the Science Platform) to display on the JupyterLab server side and provides the already-calculated string to a much more minimalist JavaScript extension.
That base hostname can be retrieved on the JupyterLab server side from Repertoire.

Nublado tutorials
^^^^^^^^^^^^^^^^^

Although it is somewhat unrelated to the rest of the service discovery problem, Repertoire should also provide the URL to the tutorials repository for that instance of the Science Platform (if there is one), and provide structure information for that repository that can be used by the ``tutorials`` extension to construct a menu.

.. _implementation-helpers:

Python helper library
---------------------

The requirement use cases can be simplified to two: a standard, flexible API that can be used with an arbitrary service that may or may not have a dedicated client (except for the Butler as discussed below), and per-service APIs that return an initialized client for that service.

For the first case, provide two functions via lsst-rsp_ that are implemented for all services except the Butler:

``get_rsp_service_url(SERVICE, DATASET)``
    Get the base URL of the API for SERVICE when querying or working with DATASET.

``get_rsp_httpx_client(SERVICE, DATASET)``
    Get an HTTPX_ client configured with the user's token and the base URL for SERVICE when querying or working with DATASET.
    HTTPX is consistent with other Rubin Science Platform code and allows easy extension to async, but returning a Requests_ client may be more standard.
    This may warrant some further thought.

.. _HTTPX: https://www.python-httpx.org/
.. _Requests: https://docs.python-requests.org/en/latest/index.html

.. note::

   It's not clear whether the name of the service should be an enum or an arbitrary string.
   The advantage of an enum is that it discourages putting random things in service discovery and exposing them to a user API, thus unintentionally creating a backwards-compatibility burden.
   It also allows quick identification of typos.
   The drawback is a lack of flexibility: A change to the lsst-rsp library would be required for the addition of each new service, and older images with older installed libraries would not be able to get information about newer services.
   This is a similar problem to the question of how to construct the Repertoire service discovery model: whether to use a mapping of arbitrary service names to URLs or to make the API model aware of the names of the specific services that can be found via service discovery if they are supported.

In addition, for services with a good client that we want to support and encourage, there will be additional helper functions that return a pre-configured client for that service.
For example:

``get_rsp_siav2_service(DATASET)``
    Return a PyVO SIAv2 client configured for the Rubin Science Platform and the given dataset.

``get_rsp_tap_service(DATASET)``
    Return a PyVO TAP client configured for the Rubin Science Platform and the given dataset.

This proposal uses a standard ``get_rsp_`` prefix for all functions at the cost of having to deprecate all of the existing helpers.
This would be at least the third round of deprecation.
An alternative would be to keep the ``get_tap_service`` and ``get_siav2_service`` method names and continue with that pattern for other services, avoiding yet another deprecation round.

The Butler client is a special case: It already expects a dataset name and uses the environment variable ``DAF_BUTLER_REPOSITORY_INDEX`` to configure itself.
Ideally it should eventually switch to using service discovery and the Repertoire library, but Repertoire only supports the Butler server, not the local Butler used in some Phalanx environments.
For the time being, we won't change how Butler does service discovery, and we will live with the duplication of dataset information between the Butler configuration and Repertoire.

``EXTERNAL_INSTANCE_URL``
^^^^^^^^^^^^^^^^^^^^^^^^^

The historical exposure of the base URL of the Science Platform to the notebook environment as ``EXTERNAL_INSTANCE_URL`` is a bit of a nightmare to deal with.
The premises behind that environment variable will become obsolete once the Science Platform is further restructured to use multiple domains.
It is already causing problems today with per-user subdomains for notebooks.

Ideally, we would drop this environment variable completely and expect all users to use service discovery.
Unfortunately, its use has escaped containment within code maintained by SQuaRE (as any environment variable is prone to do), so this is a backwards compatibility break.
We don't know how widely the variable is used, and it may be present in T&S code for summit operations.

One compromise option would be to remove it from the Nublado controller and drop it from platforms for external users, where its semantics will not continue to be honored due to domain separation of Science Platform components, and reintroduce it as an environment-specific variable added on internal T&S Science Platform instances.

.. _implementation-helpers-env:

Other environment variables
^^^^^^^^^^^^^^^^^^^^^^^^^^^

As much as possible, we should drop all of the other environment variables that we expose to the user's environment to provide various service discovery mechanisms.
Here is an incomplete list in addition ot ``EXTERNAL_INSTANCE_URL``:

- ``API_ROUTE``
- ``FIREFLY_ROUTE``
- ``HUB_ROUTE``
- ``RSP_SITE_TYPE``
- ``TAP_ROUTE``

Dropping those environment variables will cause old versions of the helper functions to stop working, so we may have to keep them for some time for backwards compatibility.

.. _implementation-sasquatch:

Sasquatch
---------

Repertoire will take over the InfluxDB database discovery function of Segwarides_ and extend that functionality to support discovery of all local InfluxDB databases.
For remote EFD access, it will also support retrieving, by name, the connection information of any remote EFD accessible from that environment, as well as retrieving the default (local) EFD name and connection information.

.. _Segwarides: https://github.com/lsst-sqre/segwarides

InfluxDB database discovery will be a separate authenticated API using Gafaelfawr token authentication.
InfluxDB information will not be included in the regular internal service discovery because it is not useful without the authentication credentials (which cannot be provided via the unauthenticated route) and is not (necessarily) an internal service.

Visibility of the discovery and authentication information for an InfluxDB database may be restricted by role.
The role check should be performed by Repertoire itself to avoid the unnecessarily complex ingress configuration required for Gafaelfawr to perform the role check.

For the time being, we will continue to use username and password authentication for the connection to the underlying InfluxDB instance.
Repertoire will return a static read-only username and password on request as part of the response to the authenticated service discovery request.

For remote EFD access, we will have to duplicate the password information for every EFD in each environment from which EFD connections are supported.
For the time being, this will require manual copying of that password between the 1Password vaults for the various environments.
The connection information, apart from the password, is not secret and can be recorded in the :file:`values.yaml` file for the Repertoire Helm chart.

.. note::

   We should attempt to limit cross-environment access to InfluxDB databases as much as possible.
   It is complex to manage in our security model.

Once this is deployed, Segwarides will be permanently retired, so all existing uses of Segwarides will need to switch to Repertoire.

.. _implementation-tap-schema:

TAP schemas and associated metadata
-----------------------------------

The goals for a replacement approach are:

- Configure the current versions of all TAP schemas in only one place.
- Serve associated metadata that matches the version of the TAP schema being deployed.
- Move the TAP schema database out of an ad hoc MySQL container and into the underlying infrastructure database.
  For data.lsst.cloud, this means Cloud SQL, matching how the UWS database is now handled.

TAP schema management conceptually could be separated from the other aspects of service discovery discussed here.
For the time being, I've chosen to combine them.
Primarily this is for convenience and to minimize the number of Phalanx services we need to deploy, although there is also some overlap in data and concepts.
Both systems have to track what data sets are available, for example.

The new proposed design is as follows:

#. The Helm chart for Repertoire, via the normal Phalanx :file:`values.yaml` mechanism, specifies the version of the TAP schema to use for each TAP service.
#. On startup, Repertoire will retrieve the TAP schema and associated metadata from the sdm_schemas_ repository and cache it.
#. On startup, Repertoire will convert the TAP schema to the necessary database data for the ``TAP_SCHEMA`` database for each TAP server and, if necessary, modify the corresponding infrastructure database contents to match.
   This will require using Felis_ as a library to generate the data.
#. The TAP servers and the datalinker_ server (or its successor when the microservices are moved to other services) will retrieve the metadata they use from Repertoire's API.
   This data may be cached locally but should be refreshed periodically so that those services do not require a restart to pick up new data.

.. _Felis: https://felis.lsst.io/

To satisfy the third point, Repertoire will have a read/write account to the underlying infrastructure database that allows it to replace the contents of the ``TAP_SCHEMA`` database.
The TAP servers will use a separate read-only account.
Permissions setup will also be done by Repertoire, as the service that manages the ``TAP_SCHEMA`` database.
Account creation will be done via Terraform in https://github.com/lsst/idf_deploy.

.. _implementation-docs:

Documentation
-------------

Phalanx_ has all of the information required to determine the URLs of every configured service in every Phalanx environment, since the source for all of Repertoire's information comes from its Phalanx configuration.
To make this information available for documentation sites, it needs to be published statically.

Repertoire will be designed as a service wrapping a library included in ``rubin-repertoire``.
That library will accept, as input, the merged Repertoire configuration for a given Phalanx environment, and will return the service discovery results.

As part of its documentation build process, Phalanx will use that library to generate a JSON file containing the static service discovery data for each environment.
That JSON file will be published as part of the Phalanx documentation site at a known URL.
Other documentation sites, such as rsp.lsst.io_ and Sasquatch_, can then retrieve that file during build time and use it as input to pages that provide information about different Phalanx environments.

.. _rsp.lsst.io: https://rsp.lsst.io/
.. _Sasquatch: https://sasquatch.lsst.io/

The username and password information for InfluxDB databases will not be included in this JSON file, since the JSON file is available without authentication and its contents will be incorporated into public documentation.

.. _dynamic:

Dynamic service discovery
=========================

There are two main ways to handle service discovery, static and dynamic.

Static service discovery uses only information injected via Phalanx to determine the expected list of services and their URLs.
This will always reflect intended reality and is much easier to implement, but is less suited for general use of Phalanx for purposes other than the Rubin Science Platform.
Adding new recognized services to a static scheme requires at least Phalanx changes to the Repertoire service and possibly code changes to Repertoire itself.

Dynamic service discovery discovers the available services at execution time.
For Kubernetes environments, this can reuee the mechanism Kubernetes itself has for service discovery: Kubernetes resources created and managed by each application.
In the case of Phalanx, the logical Kubernetes object to use for this purpose is ``GafaelfawrIngress``, since we create at least one ``GafaelfawrIngress`` for every HTTP-accessible service in order to enforce authentication and authorization rules.
The ``GafaelfawrIngress`` resource could be supplemented with additional configuration parameters describing service discovery information, such as datasets served by that ingress.
The drawback of dynamic discovery is that if the resource underlying it goes missing, this error is potentially indistinguishable from an intentional configuration omitting that service, which could result in services dynamically reconfiguring themselves for the lack of a service instead of reporting errors.

The plan for Repertoire is to start with static service discovery, since this is simpler and a strict improvement over the existing Phalanx approach.
Then, once that's working, we will add dynamic service discovery to support easy addition of new services not recognized by the Repertoire code base.
Repertoire can then compare the static configuration to the results of dynamic discovery and send alerts when they don't match.

Implementation of dynamic service discovery will probably be done via Gafaelfawr's Kubernetes operator, since it already has to scan all ``GafaelfawrIngress`` resources in a given Phalanx deployment.

Code organization and Python versions
=====================================

Repertoire will be maintained as a combined server and client in a vertical monorepo (see :sqr:`075`).
The client must be available as a regular Python package on PyPI so that it can be used as a dependency of other services.

To support the :ref:`documentation use case <use-case-docs>`, the client package will also provide the logic to construct the list of services and endpoints based on a configuration.
This will allow the client to provide a library that can be called during the Phalanx documentation build process to construct the JSON file published with the Phalanx documentation.
See :ref:`implementation-docs` for more information.

Minimum Python version
----------------------

The Repertoire client must be callable inside the kernel of a Nublado notebook, so it must be installable into the Python environment used by that kernel.
This means that it must be compatible with the Python version used by the Science Pipelines stack, which usually lags considerably behind the Python version used for other Phalanx services.
Since most of the logic will live in the client, this means Repertoire as a whole, unlike other Phalanx services, will need to support the Python version used by the current Science Pipelines release.
For example, at the time this was written Phalanx services are using Python 3.13, but the Science Pipelines stack is still using Python 3.12.

This has an annoying implication for any client returned by the Repertoire client, such as a smart, model-aware client for services like Gafaelfawr_ or Wobbly_.
A simple implementation would require all of those clients to support the Science Pipelines version of Python as well, thus spreading the requirement to support older versions of Python.
That, in turn, would create problems for the corresponding services.
For example, ideally one would maintain the client and server together in a single `uv workspace`_, but doing so limits testing to the intersection of supported Python versions, thus either forcing the server to support old Python versions or preventing testing of the client on the older Python version.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/
.. _Wobbly: https://github.com/lsst-sqre/wobbly
.. _uv workspace: https://docs.astral.sh/uv/concepts/projects/workspaces/

Our strong preference when supporting Phalanx services is to routinely upgrade the minimum Python version to the latest release and make aggressive use of new features.
We therefore want to minimize the amount of code that has to maintain compatibility with older Python versions.

Therefore, we will provide a separate version of a Repertoire client, with limited functionality, that is intended for use within Nublado notebooks.
This version would not support returning full clients for any of our internal services, only the clients we expect to be used by astronomers.
This client would be maintained as a separate project, distinct from the regular Repertoire client used by other services and by the Repertoire server.

Appendix: State as of 2025-07-31
================================

Below is a description of the service discovery approach of the Rubin Science Platform before the start of work.
It is included to aid in understanding what problems we were attempting to solve.

Internal service discovery
--------------------------

Currently, the Rubin Science Platform largely runs under a single domain, which exceptions only for Nublado on some environments.
Many applications need to know the URLs of other applications running on the same Science Platform instance.
Today, this is mostly done by using Phalanx mechanisms to inject ``global.baseUrl`` and ``global.host`` into every Helm chart, and to then add known URL paths to that base URL to construct internal URLs.
Sometimes this is done in Phalanx (usually in Helm templates), and sometimes it is done in code, either service code or client libraries.

A mapping from dataset names to Butler configuration URLs is injected into every application that uses a Butler client via the ``global.butlerServerRepositories`` value, which is a base64-encoded representation of that mapping suitable for decoding and then setting as the value for the ``DAF_BUTLER_REPOSITORIES`` environment variable.
That variable, in turn, is read by the Butler client library during initialization.

VO service discovery
--------------------

The Rubin Science Platform currently does not provide any meaningful VO service discovery.
The Portal uses configured URLs to the components that it needs to talk to, similar to the approach for internal service discovery.
Some services (vo-cutouts, the TAP servers, SIAv2) provide VOSI capabilities endpoints, but there is no collection of that information into VOResource records.

HiPS list generation
--------------------

datalinker is currently responsible for assembling the HiPS list file for each dataset from a list of paths and a base URL.
The list of datasets, base URLs, and paths are included in the per-environment Helm chart configuration.
datalinker exposes two APIs, one at ``/api/hips/list`` (legacy, including only a single dataset) and one per dataset at :samp:`/api/hips/v2/{dataset}/list` (preferred).

Nublado extensions
------------------

Currently, all of the server-side Nublado extension code except for the Firefly extension uses ``EXTERNAL_INSTANCE_URL`` to get the top-level URL of the Rubin Science Platform instance and then constructs other URLs from that base URL.
This environment variable is injected by the Nublado controller into the enivonrment variables of any spawned lab.
All other paths are hard-coded relative to that setting.

The Firefly extension is configured with the environment variable ``FIREFLY_URL``, set by ``lsst.rsp.startup``.

Information about the structure of the Nublado tutorials repository is currently maintained by a cron job run by the Nublado Phalanx application that writes into a shared NFS directory.
That information is then cached locally in the user's home directory and provided to the extension by a JupyterLab server-side handler.

Python helper library
---------------------

The current implementation of this helper library lives in lsst-rsp_ and is supported by various environment variables set either by the Nublado controller directly (``EXTERNAL_INSTANCE_URL``) or by Helm chart configuration of the controller (all of the other ones listed in :ref:`implementation-helpers-env`).

This module currently provides the following functions that do service discovery, directly or indirectly:

- ``get_catalog`` (deprecated)
- ``get_datalink_result``
- ``get_obstap_service`` (deprecated)
- ``get_query_history``
- ``get_siav2_service``
- ``get_tap_service``
- ``retrieve_query``

The functions ``lsst.rsp.utils.get_service_url`` and ``lsst.rsp.utils.get_pyvo_auth`` both do some service discovery.
They aren't exported from the top level but aren't marked as private functions, so may have leaked into user code.

lsst-rsp also provides the ``RSPClient`` class, which is a subclass of ``httpx.AsyncClient`` that preconfigures authentication and a suitable base URL for a given service, based on ``EXTERNAL_INSTANCE_URL``.

Sasquatch
---------

Currently, the Segwarides_ service running in Roundtable_ provides both discovery and authentication credentials for all EFD instances.
A client, at any Science Platform (or outside of any of them), tells Segwarides what EFD they want to connect to, and Segwarides returns the connection and authentication information for that EFD instance.
Normally, this is done via lsst-efd-client_.

.. _Segwarides: https://github.com/lsst-sqre/segwarides
.. _Roundtable: https://roundtable.lsst.io/
.. _lsst-efd-client: https://efd-client.lsst.io/

This approach has two problems.
First, it requires running a global Segwarides service, which in turn creates cross-domain authentication issues that we are currently ignoring.
Second, this architecture does not support the desired property of directing the user to the local instance by default, since it doesn't know which instance is local.

We do not currently provide service discovery for non-EFD InfluxDB databases.

TAP schemas and associated metadata
-----------------------------------

Schemas for the Rubin Science Platform are managed using Felis_ from metadata stored in sdm_schemas_.
As part of the build process of that repository, Felis generates MySQL tables holding the schema information and bundles them into a Docker image for a MySQL server that contains only the ``TAP_SCHEMA`` table.
This server is deployed in Phalanx_ as the tap-schema application, and the TAP server is configured to point to it.

.. _Phalanx: https://phalanx.lsst.io/

This approach has multiple serious problems:

- It is a waste of resources to run an entire MySQL server to serve a small handful of static tables, let alone a separate MySQL server per TAP server.
- Authentication to this database is handled poorly.
- The MySQL image used for those containers is not systematically kept up to date.
- The Science Platform has a policy of not including database servers (except for the special case of InfluxDB) for production instances, instead requiring all database services to be provided and maintained by the infrastructure provider.

This approach does have the advantage that these containers are built for every pull request as well as every tag, and thus satisfy the requirement that unreleased schemas can be easily tested in non-production environments.

Additional TAP metadata
^^^^^^^^^^^^^^^^^^^^^^^

Currently, each release of sdm_schemas_ also generates, via GitHub Actions, two additional release artifacts: :file:`datalink-snippets.zip` and :file:`datalink-columns.zip`.
The TAP service downloads the former on restart from GitHub and uses it to control which DataLink records are added to result tables.
The datalinker_ service downloads the latter on restart and parses it for information about the TAP schema.

This approach requires manual configuration changes to both the TAP and datalinker services in Phalanx each time there is a new relevant sdm_schemas release.
Doing this via Helm chart configuration also makes it awkward to use different versions of the data in different environments.
If someone forgets to change the versions of the data downloaded by those services, they may also use metadata that is out of sync with the ``TAP_SCHEMA`` metadata provided by the tap-schema application.
