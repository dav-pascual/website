Overview
========

FreeIPA servers currently use a highly privileged *RA Agent* certificate
to perform operations in Dogtag (such as certificate issues, managing
profiles and managing lightweight CAs). This lack of privilege
separation has caused numerous security issues i.e. where the IPA
framework has failed to appropriately authorise use of the RA Agent
credential.

External authentication support for Dogtag is being implemented
(https://fedorahosted.org/pki/ticket/1359). This work will allow
externally-authenticated principals to perform operations in Dogtag,
subject to permission checks against an authorization plugin determined
by the realm of the principal. "Externally-authenticated" means that an
authenticating proxy (e.g. Apache with mod_auth_gssapi and
mod_lookup_identity) has authenticated the user and conveyed the
authenticated principal's name and groups to Dogtag in the request
environment.

This design describes how FreeIPA will move away from using the
privileged RA Agent credential, to using GSS-API authentication with
proxy tickets (i.e. operating with the privileges of the authenticated
principal).



Use Cases
=========

Improved security through privilege separation. Dogtag will enforce
access controls defined by FreeIPA.

Moving CA ACL enforcement and certificate request validation logic to
Dogtag is a first step towards having Dogtag populate a certificate with
information retrieved directly from the FreeIPA directory (other other
resources), instead of relying on users crafting a CSR that is "just
right" for a particular profile, improving usability and expanding the
number of certificate use cases that FreeIPA can satisfy.



Feature Management
==================

As discussed below, a domain level bump is required to switch from using
the RA Agent certificate to using GSS-API and proxy tickets.

No other administrator intervention is expected, and there are no
user-visible changes associated with this effort.

Design
======

.. _overview_of_dogtag_external_authentication_support:

Overview of Dogtag external authentication support
--------------------------------------------------

Design page: http://pki.fedoraproject.org/wiki/GSS-API_authentication

The most important changes that affect FreeIPA are discussed in the
following subsections.

.. _freeipa_must_manage_dogtag_acls_that_refer_to_freeipa_entities:

FreeIPA must manage Dogtag ACLs that refer to FreeIPA entities
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When authorising operations for externally-authenticated principals,
Dogtag looks up an ``IAuthzManager`` plugin instance based on the realm
of the principal. FreeIPA must configure this plugin instance, and must
also manage the ACLs that is uses.

FreeIPA already does limited Dogtag ACL management to allow the RA Agent
to perform required operations (these ACLs are loaded by the default
``IAuthzManager`` plugin. Now, FreeIPA will need to manage a separate
set of ACLs that will be used by the new authz plugin instance. These
ACLs will refer to principals and groups defined in the IPA domain,
rather than "internal" Dogtag users and groups.

Technical details about these ACLs and management thereof are provided
later in this document.

.. _freeipa_must_configure_dogtag_to_perform_cert_request_validation_and_authorisation:

FreeIPA must configure Dogtag to perform cert request validation and authorisation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Currently, certificates are requested with the RA Agent credential.
FreeIPA-defined certificate profiles are configured to use the
``raCertAuth`` plugin which enables immediate issuance of the
certificate when the operator is the RA Agent.

In moving to GSS-API authentication, the certificate will now be
requested with a proxy credential for the FreeIPA principal. (It should
also be noted that a principal that is logged onto a FreeIPA server with
a CA instance will be able to bypass the IPA framework and issue a
certificate request *directly* to Dogtag). Accordingly, the
authorisation (chiefly, CA ACLs) and request validation checks that are
currently performed in the IPA framework prior to conveying the
certificate request to Dogtag with RA Agent credential must now be
executed within Dogtag itself.

Dogtag will provide the new ``ExternalProcessConstraint`` profile
component, which can be used for this purpose. FreeIPA must configure an
instance of this program that executes a program provided by FreeIPA,
that will check CA ACLs and validate the CSR. FreeIPA must add the
``ExternalProcessConstraint`` instance to all FreeIPA-managed profiles
(including custom profiles).

Technical details about the validation program and profile configuration
are provided later.

.. _migration_considerations:

Migration considerations
------------------------

Existing deployments of FreeIPA cannot be updated to a new version that
supports Dogtag GSS-API authentication all at once. There is inevitably
a period where interoperation of old and new server versions is
required. This presents some challenges.

It will help to first describe the various topologies that can occur.
Here, an *old server/CA* is at a version that does not support GSS-API
authentication to Dogtag, and a *new server/CA* is at a version that
does support GSS-API authentication, whether or not it is actually
configured to use it.

-  **New server communicating with new CA**. OK to use GSS-API
   authentication.
-  **Old server communicating with new CA**. RA Agent certificate must
   be used (server does not support GSS-API auth to Dogtag).
-  **New server communicating with old CA**. RA Agent certificate must
   be used (CA does not support external authentication).

For normal administrative operations it is obvious that until all
servers in the topology are new servers, servers must retain the RA
Agent certificate. In a mixed topology, new servers talking to new CAs
could use either RA Agent cert auth, or SPNEGO with a proxy ticket. The
latter is preferable security-wise, but we will use the former for
reasons discussed below.

Turning our attention to certificate requests, observe that because
Dogtag certificate profile configurations are stored in LDAP (and
therefore replicated), upgrading FreeIPA-managed profile configurations
(to add the ``ExternalProcessConstraint``) cannot occur until all
servers in the topology are new servers (because
``ExternalProcessConstraint`` does not exist in older versions of
Dogtag). Therefore, **in a mixed topology we cannot use SPNEGO
authentication for certificate requests, or even upgrade profile
configurations**. Upgrading the profiles must be deferred until there
are no old servers in the topology.

.. _domain_level_2:

Domain Level 2
~~~~~~~~~~~~~~

The way to solve this challenge is to introduce a new *domain level* (at
time of writing, this will be DL 2). It is not possible to upgrade to a
particular domain level unless all servers in the topology support that
domain level, so by introducing a new domain we can enforce the
requirement that all servers in the topology support Dogtag GSS-API
authentication before we start using it.

After upgrading to the new server version, but *before* setting domain
level = 2:

-  All configuration changes described in this document *except* profile
   configuration changes can be applied (and will have been applied
   during server update).
-  Accordingly Dogtag (when accessed via the Apache frontend) will
   support SPNEGO authentication, but communications between the IPA
   framework and Dogtag (which are mediated by the backends defined in
   ``ipaserver.plugins.dogtag``) will continue to use the RA Agent
   credential.

After setting domain level = 2:

-  IPA-managed profile configurations shall be updated to add the
   ``ExternalProcessConstraint``. This only needs to be done once
   (because of LDAP profile replication). Restart is not required.
   Preferably it would be performed automatically. **QUESTION**: is
   there a way to trigger this sort of behaviour upon DL change? If not,
   can it be put into ``domainlevel_set``?
-  The Dogtag backends plugin (``ipaserver.plugins.dogtag``) shall begin
   using SPNEGO authentication with proxy tickets.
-  The RA Agent cert can be removed from each server. It is preferable
   for this to occur automatically. It could be deferred until the next
   execution of ``ipa-server-upgrade`` which, if DL >= 2 and RA Agent
   cert is present, removes the cert and associated key.
-  The RA Agent user account and associated ACLs can be removed from the
   Dogtag database. (This is not an essential step; more of a tidy-up).
-  Replica installation will not attempt to install the RA Agent cert
   (it is not needed and cannot be assumed to exist).

New installations (which will automatically be in DL 2) will no longer
create the RA Agent account or certificates.

.. _server_configuration_changes:

Server configuration changes
----------------------------

SSSD
~~~~

The ``sssd-dbus`` package, which provides the *InfoPipe* D-Bus
responder, is required.

SSSD on servers must be configured to allow *mod_lookup_identity* to
query a principal's ``memberOf`` attribute.

Example ``/etc/sssd/sssd.conf`` configuration (indicative only):

::

   [sssd]
   services = nss, sudo, pam, ssh, ifp
   ...

   [domain/EXAMPLE.COM]
   ...
   ldap_user_extra_attrs = roles:memberOf

   [ifp]
   allowed_uids = apache
   user_attributes = +roles

The attribute is exposed under the name ``roles``. The name ``memberOf``
seems to have special treatment and does not result in the required
behaviour.

SELinux
~~~~~~~

SELinux must be configured to allow Apache to query the SSSD InfoPipe.

::

   $ sudo setsebool -P httpd_dbus_sssd 1

httpd
~~~~~

The ``mod_lookup_identity`` package is required.

``/etc/httpd/conf.d/ipa-pki-proxy.conf`` shall be updated to perform
SPNEGO authentication when a client requests Dogtag resources.
``mod_lookup_identity`` shall populate the AJP request environment with
groups and permissions of the authenticated principal (if any).

Example (indicative only):

::

   &lt;If &quot;%{QUERY_STRING} =~ /\bgssapi=/&quot;&gt;
     AuthType GSSAPI
     AuthName &quot;Kerberos Login&quot;
     GssapiCredStore keytab:/etc/httpd/conf/ipa.keytab
     GssapiCredStore client_keytab:/etc/httpd/conf/ipa.keytab
     GssapiDelegCcacheDir /var/run/httpd/ipa/clientcaches
     GssapiUseS4U2Proxy on
     GssapiAllowedMech krb5
     Require valid-user
     LookupUserAttrIter roles +AJP_REMOTE_USER_GROUP
   &lt;/If&gt;

A query string is used to activate SPNEGO authentication because, due
the version interoperability requirements discussed above, this
configuration must be able to support both SPNEGO authentication and the
legacy certificate authentication method. Requiring the query string
allows requests that do not contain it to bypass SPNEGO authentication
and proceed the old-fashioned way.

This imposes a burden on the client: it must provide the query string if
it wishes to use SPNEGO authentication. This is not a problem because
the only client of significance is the IPA framework, which we control.

.. _client_certificate:

Client certificate
^^^^^^^^^^^^^^^^^^

The ``NSSVerifyClient require`` directive shall be relaxed to
``NSSVerifyClient optional``. This is needed so that GSS-API
authentication can be used for affected resources. Codepaths that are
configured to present a certificate will still do so.

.. _pki_tomcatd:

``pki-tomcatd``
~~~~~~~~~~~~~~~

The ``pki-tomcatd`` deployment must be updated to accept external
authentication. In ``/etc/pki/pki-tomcat/server.xml``:

::

   &lt;Connector port=&quot;8009&quot;
     protocol=&quot;AJP/1.3&quot;
     tomcatAuthentication=&quot;false&quot;  &lt;!-- add this attribute --&gt;
     redirectPort=&quot;8443&quot;
     address=&quot;localhost&quot; /&gt;

``CS.cfg``
~~~~~~~~~~

``/etc/pki/pki-tomcat/{ca,kra}/CS.cfg`` must be updated to define an
``IAuthzManager`` plugin instance for the FreeIPA realm.

Directives to be added:

::

   authz.instance.IPAAuthz.pluginName=DirAclAuthz
   authz.instance.IPAAuthz.ldap=internaldb
   authz.instance.IPAAuthz.searchBase=cn=IPA,cn=aclResources
   authz.instance.IPAAuthz.realm=${ACTUAL_REALM}

.. _dogtag_acl_management:

Dogtag ACL management
---------------------

Previously, FreeIPA added attribute values to the main Dogtag ACLs entry
(``cn=aclResources,o=ipaca``) to allow the RA Agent to perform required
operations.

Now, FreeIPA will manage ACLs in a separate entry that will be read by
the ``IAuthzManager`` for the IPA realm. These ACLs use the standard
Dogtag ACL syntax but will refer to IPA users (or other principal
names), groups and permissions, rather than "internal" Dogtag users and
groups. The entry shall be:

::

   cn=IPA.LOCAL,cn=aclResources,o=ipaca

ACLs may need to allow host principals that are members of the
``ipaservers`` group to perform some operations (e.g. profile
management) during installation and upgrade.

**TODO**: detail the various operations and provide example ACLs.

.. _adding_externalprocessconstraint_to_profile_configurations:

Adding ``ExternalProcessConstraint`` to profile configurations
--------------------------------------------------------------

**TODO** describe when and how this will occur

.. _the_ipa_pki_validate_cert_request_program:

The ``ipa-pki-validate-cert-request`` program
---------------------------------------------

The program to be executed by ``ExternalProcessConstraint`` for
FreeIPA-managed profiles shall be installed at
``/usr/libexec/ipa/ipa-pki-validate-cert-request``.

It will be a Python program whose logic consists primarily of existing
code for checking CA ACLs and validating CSR contents against the IPA
directory. (Refactorings shall occur accordingly). Other behaviour of
the program shall be to unmarshall data from the execution environment
and output the result in the required manner.

The program must be able to connect to the database to look up
information required to authorise and validate the request, including CA
ACLs and virtual operation permissions. Therefore, the bind principal
**must have permission** to read relevant entries, and in the case of
virtual operations, to execute the ``GetEffectiveRights`` control
against relevant permissions.

Reading effective rights of a given user on an entry can only be done by
*cn=Directory Manager* or by that user themselves. The implication is
that ``ipa-pki-validate-cert-request`` must bind as the *operator*
principal who is executing the certificate request. Therefore, a proxy
ticket for the operator must be acquired and used when talking back to
the FreeIPA directory. Apache must be configured to give Dogtag (i.e.
``pkiuser``) access to a client credential cache for this purpose.

**TODO** the precise program contract w.r.t. environment, args, input,
output, exit status, etc, is yet to be finalised.

Implementation
==============

.. _dogtag_client_credential_cache:

Dogtag client credential cache
------------------------------

The ``ipa-pki-validate-cert-request`` program must use a proxy ticket to
operate on behalf of the authenticated user when talking back to
FreeIPA. *mod_auth_gssapi* must be configured to establish a credential
cache that can be read by ``pkiuser``.

*mod_auth_gssapi* itself run as the ``apache`` user. It is not
appropriate to add ``pkiuser`` to the ``apache`` group, or vice versa,
in order for *mod_auth_gssapi* to write credential caches that are
readable by ``pkiuser``. Instead, a simple way to accomplish this is to
have *mod_auth_gssapi* write a **world-readable** ccache inside a
directory that is readable only by ``apache`` and ``pkiuser``.

The containing directory shall be ``/var/run/pki/clientcaches/`` with
ownership ``apache:pkiuser`` and mode ``0640``. The credential caches
created therein shall have mode ``0644``. The following ``httpd``
configuration directives are involved:

::

   GssapiDelegCcacheDir /var/run/pki/clientcaches
   GssapiDelegCcachePerms mode:0644

Upgrade
=======

Explicit upgrade steps that will be required include:

-  Update SSSD config (described above)
-  ``setsebool -P httpd_dbus_sssd 1`` (described above)
-  Update ``/etc/pki/pki-tomcat/server.xml`` (described above)
-  Add ``ExternalAuthenticationValve`` to
   ``/etc/pki/pki-tomcat/Catalina/localhost/ca.xml``.
-  Update ``CS.cfg`` files (described above)
-  Write Dogtag ACLs for the FreeIPA realm

Configuration changes that will automatically occur during upgrade
include:

-  Update ``ipa-pki-proxy.conf`` (described above; updating the template
   is sufficient to effect this change during upgrade).



How to Use
==========

To switch an existing deployment from RA Agent certificate
authentication to SPNEGO proxy ticket authentication:

#. Ensure all servers in the topology are at the new version
#. Execute ``ipa domainlevel-set 2``



Test Plan
=========
