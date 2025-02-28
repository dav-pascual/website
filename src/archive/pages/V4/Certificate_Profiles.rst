Overview
========

FreeIPA currently only supports host and service certificates and has a
single, hard-coded certificate profile. This proposal introduces the
ability to define new certificate profiles and control which subject
principals or principal types (users, hosts or services) they can be
used for.



Use Cases
=========

.. _custom_extension_values:

Custom extension values
-----------------------

The current certificate profile sets both TLS Server Authentication and
TLS Client Authentication in the *Extended Key Usage* extension. This is
not appropriate for most uses. Profile support will allow the
appropriate profile(s) to be defined.

Furthermore, wherever there is a use case that requires specific values
in certificate fields or extensions, it will be possible to import a
profile that supports that use case as long as Dogtag supports the
certificate extension(s) to be included.

.. _dnp3_sav5:

DNP3 SAv5
---------

The DNP3 Secure Authentication version 5 (SAv5) standard uses the IEC
62351-8 certificate authentication to carry authorization data for the
DNP3 smart-grid technology. A specialised certificate profile would be
needed to support DNP3.

Design
======

Profiles will be stored in Dogtag. A small amount of metadata will be
stored in FreeIPA's directory to track these profiles, provide a
description and store whether certificates issued using the profile will
be stored in the FreeIPA directory.

IPA must be modified to respect the profile parameter in requests from
Certmonger (currently ignored).

Rich profile management (use of a command-line tool or Web UI to build
new profiles for use with FreeIPA, rather than the presuppose the
existence of a profile) can be implemented on top of the basic profiles
support, if there is demand. At a minimum, there should be tutorials and
improved documentation in Dogtag for how to define certificate profiles.

Terminology
-----------

*included profile*
   Any profile that is shipped as part of FreeIPA and available in a
   default installation.
*custom profile*
   Any profile that has been imported by an administrator.
*FreeIPA-managed profile*
   A profile that was added via FreeIPA and has an associated object in
   the FreeIPA directory, containing FreeIPA-specific configuration,
   distinguishing it from other profiles in Dogtag.

.. _profile_backend:

Profile backend
---------------

A new backend will be implemented to provide the profile management
behaviours while abstracting the Dogtag integration. The profile
management *plugin* shall invoke the profile backend to do the work of
communicating with Dogtag.

.. _profile_formats:

Profile formats
~~~~~~~~~~~~~~~

There are two profile formats used by Dogtag: an XML representation, and
the "raw" property list format which is also (at the current time) the
internal storage format. Initial work will focus on the raw format, but
it should be simple to distinguish between and support both formats.

.. _listing_profiles:

Listing profiles
~~~~~~~~~~~~~~~~

The list of all Dogtag profiles is retrieved via the Dogtag REST API:

::

   GET /ca/rest/profiles

.. _profile_import:

Profile import
~~~~~~~~~~~~~~

(This section is about importing new profiles individually. For initial
import of profiles during installation or upgrade, see the
**Configuration** and **Upgrade** sections.)

New profiles can be imported via CLI (specify profile filename) or Web
UI (paste file content). Interactive "profile builder" functionality is
a future feature (see Dogtag ticket
`#1331 <https://fedorahosted.org/pki/ticket/1331>`__.)

Having obtained the profile content, FreeIPA will import import the
profile into Dogtag using the Dogtag REST API:

::

   PUT /ca/rest/profiles/&lt;profileId&gt;       (XML format)
   PUT /ca/rest/profiles/&lt;profileId&gt;/raw   (raw format)

Failure modes:

-  Profile ID already in use
-  Bad profile content

.. _retrieve_profile:

Retrieve profile
~~~~~~~~~~~~~~~~

Profile data can be retrieved from Dogtag using the REST API:

::

   GET /ca/rest/profiles/&lt;profileId&gt;       (XML format)
   GET /ca/rest/profiles/&lt;profileId&gt;/raw   (raw format)

The XML or property list (whatever is used) can be parsed to determine
name, enabled/disabled state, and other data. It is not an initial
requirement that FreeIPA provide a detailed breakdown of the profile
(inputs, policy constraints and defaults, etc), but the basic
information should be available.

Failure modes:

-  Profile ID unknown

.. _delete_profile:

Delete profile
~~~~~~~~~~~~~~

Profiles can be deleted from Dogtag using the REST API:

::

   DELETE /ca/rest/profiles/&lt;profileId&gt;

Failure modes:

-  Profile ID unknown
-  Profile enabled (profiles must be disabled before deletion)

If a profile is enabled and a FreeIPA admin attempts to delete it, we
shall raise ``StillActive`` or a similar exception.

.. _enabledisable_profile:

Enable/disable profile
~~~~~~~~~~~~~~~~~~~~~~

Enabling or disabling a profile in Dogtag is accomplished via the REST
API:

::

   POST /ca/rest/profiles/&lt;profileId&gt;?action=enable
   POST /ca/rest/profiles/&lt;profileId&gt;?action=disable

Failure modes:

-  Profile ID unknown
-  Profile already enabled/disabled

It may be useful to record the enabled/disabled state of a profile in
the FreeIPA directory, so that the state is visible and decisions can be
made based on the profile state without requiring a round-trip to Dogtag
to find out and to avoid blind attempts of operations that could fail
according to profile enabled/disabled state (e.g. profile deletion).

.. _certificate_profiles_plugin:

Certificate Profiles plugin
---------------------------

The ``certprofile`` plugin will be created for the management of FreeIPA
profiles. It will allow privileged users to import, modify or remove
FreeIPA-managed profiles in Dogtag and manage the FreeIPA-specific
profile configuration.

.. _enabling_or_disabling_profiles:

Enabling or disabling profiles
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

IPA will not provide a direct way to enable or disable profiles in
Dogtag. Separate CA ACL rules will govern whether a particular profile
can be used to issue a certificate to a particular subject princpial.
These rules can be created, modified, disabled or enabled by privileged
users. See the CA ACL section below.

.. _storing_issued_certificates:

Storing issued certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Support for multiple profiles means that principals (including user
principals) may now have *multiple certificates*. The proposed schema
and implications are discussed in the `V4/User
Certificates <http://www.freeipa.org/page/V4/User_Certificates>`__
design page.

The FreeIPA profile object class includes a boolean attribute
``ipaCertProfileStoreIssued`` that controls whether certificate issued
using that profile are stored in the subject principal's
``userCertificate`` attribute. For use cases that involve issuance of
many, possibly short-lived certificates, setting this attribute to
``FALSE`` ensures that these certificates to not accumulate in the
principal's entry.

When issuing a certificate via ``ipa cert-request``, the semantics of
``ipaCertProfileStoreIssued`` is:

-  when ``TRUE``, *add* the full certificate to the userCertificate
   attribute;
-  when ``FALSE``, store nothing at all and merely deliver the issued
   certificate in the command result.

The cert-request command will be updated to act accordingly.

Permissions
~~~~~~~~~~~

The following new permissions will be added, as will the *CA
Administrator* role which is initially granted these permissions.

-  ``System: Read Certificate Profiles`` (all principals may read)
-  ``System: Import Certificate Profile``
-  ``System: Delete Certificate Profile``
-  ``System: Modify Certificate Profile``

Schema
~~~~~~

FreeIPA will store data about the certificate profiles that are managed
via FreeIPA (including the *included profiles*). This will:

-  enable fast query of which profiles are available for FreeIPA
   principals to use (Dogtag does not have to be contacted);
-  allow storage of additional profile-related configuration that is
   specific to FreeIPA;
-  avoid exposing all of the profiles available in Dogtag to FreeIPA
   (only those managed by FreeIPA will be visible to FreeIPA users);

The data stored for each profile are:

-  Profile ID (used by Dogtag)
-  Profile summary (short description)
-  Profile certificate storage configuration (explained above)

Certificate profile entries will be stored under a new DN:
``cn=certprofiles,cn=ca,$SUFFIX``.

Schema:

::

   dn: cn=schema
   attributeTypes: ( 2.16.840.1.113730.3.8.19.1.1
     NAME 'ipaCertProfileStoreIssued'
     DESC 'Store certificates issued using this profile'
     EQUALITY booleanMatch
     SYNTAX 1.3.6.1.4.1.1466.115.121.1.7
     SINGLE-VALUE
     X-ORIGIN 'IPA v4.2' )
   objectClasses: ( 2.16.840.1.113730.3.8.19.2.1
     NAME 'ipaCertProfile'
     SUP top
     STRUCTURAL MUST ( cn $ description $ ipaCertProfileStoreIssued )
     X-ORIGIN 'IPA v4.2' )

.. _ca_acls_plugin:

CA ACLs plugin
--------------

Custom profile use cases involve the issuance of certificates for
specific, unrelated purposes. It is necessary to be able to define rules
that control which profiles can be used to issue certificates to which
principals. ACLs will be used to associate profiles, subject principals
and groups with a CA (initially just the *top-level* CA, but this
provision is made for forward-compatibility with Lightweight CAs).
Specifically:

-  An ACL can permit access to multiple CAs.
-  An ACL can permit access to multiple profiles.
-  An ACL can have multiple users, services, hosts, (user) groups and
   hostgroups associated with it.
-  The interpretation of the ACL is: *these principals (or groups) are
   permitted as the subject of certificates issued using these profiles,
   by these CAs*.

Note that the principal performing the certificate request is not
necessarily the subject principal.

See also the ``ipa caacl-*`` commands in the CLI section below.

.. _permissions_1:

Permissions
~~~~~~~~~~~

The following permissions will be created. All permissions are intially
granted to the *CA Administrator* role.

``System: Read CA ACLs``
   All may read all attributes.
``System: Add CA ACL``
   Add a new CA ACL.
``System: Delete CA ACL``
   Delete an existing CA ACL.
``System: Modify CA ACL``
   Modify the name or description, or enable/disable the CA ACL.
``System: Manage CA ACL membership``
   Manage CA, profile, user, host and service membership.

.. _schema_1:

Schema
~~~~~~

CA ACL objects shall be stored in the container
``cn=caacls,cn=ca,$SUFFIX``.

New attributes are defined for CA and profile membership and categories
("all CAs / profiles"). The ``ipaCaAcl`` object class extends
``ipaAssociation`` uses these new attributes as well as existing member
and category attributes.

Note that the ``memberCa`` and ``caCategory`` attributes are unused by
this design. They will be used by the Sub-CAs feature.

::

   attributeTypes: (2.16.840.1.113730.3.8.21.1.2
     NAME 'memberCa'
     DESC 'Reference to a CA member'
     SUP distinguishedName
     EQUALITY distinguishedNameMatch
     SYNTAX 1.3.6.1.4.1.1466.115.121.1.12
     X-ORIGIN 'IPA v4.2' )
   attributeTypes: (2.16.840.1.113730.3.8.21.1.3
     NAME 'memberProfile'
     DESC 'Reference to a certificate profile member'
     SUP distinguishedName
     EQUALITY distinguishedNameMatch
     SYNTAX 1.3.6.1.4.1.1466.115.121.1.12
     X-ORIGIN 'IPA v4.2' )
   attributeTypes: (2.16.840.1.113730.3.8.21.1.4
     NAME 'caCategory'
     DESC 'Additional classification for CAs'
     EQUALITY caseIgnoreMatch
     ORDERING caseIgnoreOrderingMatch
     SUBSTR caseIgnoreSubstringsMatch
     SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
     X-ORIGIN 'IPA v4.2' )
   attributeTypes: (2.16.840.1.113730.3.8.21.1.5
     NAME 'profileCategory'
     DESC 'Additional classification for certificate profiles'
     EQUALITY caseIgnoreMatch
     ORDERING caseIgnoreOrderingMatch
     SUBSTR caseIgnoreSubstringsMatch
     SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
     X-ORIGIN 'IPA v4.2' )
   objectClasses: (2.16.840.1.113730.3.8.21.2.2
     NAME 'ipaCaAcl'
     SUP ipaAssociation
     STRUCTURAL
       MUST cn
       MAY
         ( caCategory $ profileCategory $ userCategory $ hostCategory
         $ serviceCategory $ memberCa $ memberProfile $ memberService )
       X-ORIGIN 'IPA v4.2' )

.. _default_ca_acl:

Default CA ACL
~~~~~~~~~~~~~~

During installation we must create a default CA ACL that grants use of
caIPAserviceCert on the top-level CA to all hosts and services:

::

   dn: ipauniqueid=autogenerate,cn=caacls,cn=ca,$SUFFIX
   changetype: add
   objectclass: ipaassociation
   objectclass: ipacaacl
   ipauniqueid: autogenerate
   cn: hosts_services_caIPAserviceCert
   ipaenabledflag: TRUE
   memberprofile: cn=caIPAserviceCert,cn=certprofiles,cn=ca,$SUFFIX
   hostcategory: all
   servicecategory: all

Implementation
==============

``ipa-pki-proxy.conf`` had to be updated to allow access to the
``/ca/rest/profiles`` endpoint and to allow *either* certificate
authentication or password authentication for logging into the REST API.

As part of this feature, FreeIPA now manages its own profiles.
Previously, the default profile was provided by Dogtag itself.
(Currently, it still is, but FreeIPA overrides it, and its removal from
Dogtag should now be considered). FreeIPA profile *templates* (which
have variables that are substituted before they are imported into
Dogtag) are stored in ``/usr/share/ipa/profiles/``.

The CA ACL enforcement functions use the existing HBAC machinery from
the ``pyhbac`` module.



Feature Management
==================

UI
--

.. _profile_management_ui:

Profile management UI
~~~~~~~~~~~~~~~~~~~~~

A grid UI shall be provided that lists FreeIPA-managed profiles and
allows editing of their FreeIPA-specific configuration.

.. _ca_acl_management_ui:

CA ACL management UI
~~~~~~~~~~~~~~~~~~~~

A web UI allowing creation and management of CA ACLs will be added. It
will work similarly to the HBAC UI.

.. _certificate_management_ui:

Certificate management UI
~~~~~~~~~~~~~~~~~~~~~~~~~

There are existing UI elements for requesting a certificate for, and
displaying the certificate issued to a service principal. These aspects
of the UI must be enhanced to support multiple certificates.

For certificate requests, a drop-down list of FreeIPA-managed profiles
will be suitable for selecting a profile.

For viewing certificates, a list of certificates should be presented.
Each should identify the profile that was used to issue that
certificate, and perhaps other important information such as a
certificate fingerprint. Upon selecting a certificate the existing
dialog showing the Base-64 encoded certificate and providing options for
renewal or revocation will be shown.

CLI
---

.. _ipa_certprofile_import_id_options:

``ipa certprofile-import ID [options]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add a profile to FreeIPA and Dogtag. Profiles will be enabled by
default.

Options:

``--desc=STR``
   Brief description of this profile
``--store=BOOL``
   Whether to store certs issued using this profile
``--file=FILE``
   Name of file containing profile data (Dogtag raw format)

.. _ipa_certprofile_mod_id_options:

``ipa certprofile-mod ID [options]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--desc=STR``
   Edit the description
``--store=BOOL``
   Edit the "store issued certificates" policy for this profile
``--file=FILE``
   Name of file containing profile data (Dogtag raw format) with which
   to update Dogtag.

.. _ipa_certprofile_del_id:

``ipa certprofile-del ID``
~~~~~~~~~~~~~~~~~~~~~~~~~~

Delete the specified profile. This command will disable the profile in
Dogtag prior to deletion.

Certificates issued using the profile will be kept around; no special
action is taken in this regard.

.. _ipa_certprofile_find_criteria_options:

``ipa certprofile-find [CRITERIA] [options]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Search for Certificate Profiles.

``--id=STR``
   Profile ID
``--desc=STR``
   Brief description of the profile
``--store=BOOL``
   Search for profiles with the given store-issued setting.

Case insensitive substring or keyword match on the description is
desirable, to aid users in locating the right profile for a particular
purpose.

.. _ipa_certprofile_show_id_options:

``ipa certprofile-show ID [options]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Display the properties of a Certificate Profile.

``--out=FILE``
   Write the Dogtag profile data (Dogtag raw format) to the named file.

.. _ipa_caacl_find:

``ipa caacl-find``
~~~~~~~~~~~~~~~~~~

Search for CA ACLs.

``--name=STR``
   CA ACL name
``--desc=STR``
   Description
``--profilecat=['all']``
   Profile category. Mutually exclusive to profile members.
``--usercat=['all']``
   User category. Mutually exclusive with user members.
``--hostcat=['all']``
   Host category. Mutually exclusive with host members.
``--servicecat=['all']``
   Service category. Mutually exclusive with service members.

.. _ipa_caacl_show_name:

``ipa caacl-show NAME``
~~~~~~~~~~~~~~~~~~~~~~~

Show details of named CA ACL.

.. _ipa_caacl_add_name:

``ipa caacl-add NAME``
~~~~~~~~~~~~~~~~~~~~~~

Create a CA ACL. New CA ACLs are initially enabled.

``--desc=STR``
   Description
``--profilecat=['all']``
   Profile category. Mutually exclusive to profile members.
``--usercat=['all']``
   User category. Mutually exclusive with user members.
``--hostcat=['all']``
   Host category. Mutually exclusive with host members.
``--servicecat=['all']``
   Service category. Mutually exclusive with service members.

.. _ipa_caacl_mod_name:

``ipa caacl-mod NAME``
~~~~~~~~~~~~~~~~~~~~~~

Modify the named CA ACL.

``--desc=STR``
   Description
``--profilecat=['all']``
   Profile category. Mutually exclusive to profile members.
``--usercat=['all']``
   User category. Mutually exclusive with user members.
``--hostcat=['all']``
   Host category. Mutually exclusive with host members.
``--servicecat=['all']``
   Service category. Mutually exclusive with service members.
``--setattr``, ``--addattr``, ``--delattr``
   As per other IPA framework commands.

.. _ipa_caacl_del_name:

``ipa caacl-del NAME``
~~~~~~~~~~~~~~~~~~~~~~

Delete the CA ACL.

.. _ipa_caacl_enable_name:

``ipa caacl-enable NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~

Enable the named CA ACL.

.. _ipa_caacl_disable_name:

``ipa caacl-disable NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~

Disabled the named CA ACL.

.. _ipa_caacl_add_profile_name:

``ipa caacl-add-profile NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add profile(s) to the CA ACL.

``--certprofiles=STR``
   Certificate Profiles to add.

.. _ipa_caacl_remove_profile_name:

``ipa caacl-remove-profile NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Remove profile(s) from the CA ACL.

``--certprofiles=STR``
   Certificate Profiles to remove.

.. _ipa_caacl_add_user_name:

``ipa caacl-add-user NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--users``
   Add user(s)
``--groups``
   Add user group(s)

.. _ipa_caacl_remove_user_name:

``ipa caacl-remove-user NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--users``
   Remove user(s)
``--groups``
   Remove user group(s)

.. _ipa_caacl_add_host_name:

``ipa caacl-add-host NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--hosts``
   Add host(s)
``--hostgroups``
   Add host group(s)

.. _ipa_caacl_remove_host_name:

``ipa caacl-remove-host NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--hosts``
   Remove host(s)
``--hostgroups``
   Remove host group(s)

.. _ipa_caacl_add_service_name:

``ipa caacl-add-service NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--services``
   Add service(s)

.. _ipa_caacl_remove_service_name:

``ipa caacl-remove-service NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--services``
   Remove service(s)

.. _ipa_cert_request:

``ipa cert-request``
~~~~~~~~~~~~~~~~~~~~

Modify command to add **optional** ``--profile-id ID`` argument to
specify which profile to use. If not given, the default
``caIPAserviceCert`` profile will be used.

Configuration
-------------

FreeIPA must be deployed with the Dogtag RA in order to use these
features. No other configuration is required.

There is no configuration in FreeIPA to enable or disable profiles in
Dogtag. FreeIPA-managed profiles are automatically enabled in Dogtag
upon import.

Essential profiles (if any beyond the default set in Dogtag) will be
added and enabled on server installation. Other "pre-canned" profiles
can be introduced by FreeIPA in the future, as required.

Upgrade
=======

The upgrade process ensures that included profiles are imported and
enabled.

Dogtag instances must be configured to use LDAP-based profiles, so that
they are replicated. This involves setting
``subsystem.1.class=com.netscape.cmscore.profile.LDAPProfileSubsystem``
in Dogtag's ``CS.cfg`` and importing profiles.

.. _upgrading_default_profiles:

Upgrading default profiles
--------------------------

If an *included profile* (i.e., a profile supplied by FreeIPA) needs to
be updated, an upgrade script can call invoke the profile backend to
update it. Any changes to the behaviour of included profiles should be
adequately documented in release notes.

.. _handling_inconsistent_profiles:

Handling inconsistent profiles
------------------------------

We take a "first upgrade wins" approach - whichever replica is upgraded
first, its profiles are imported. On other replica, the presence of LDAP
profiles will be detected and no import or conflict resolution is
attempted. This behaviour must be clearly explained and administrators
who have custom profiles encouraged to check for inconsistencies prior
to upgrade.

.. _adding_default_ca_acl:

Adding default CA ACL
---------------------

On upgrade, a default CA ACL added that permits host and service
principals to use the default profile, ensuring that current
capabilities are maintained.



How to Use
==========

See
https://blog-ftweedal.rhcloud.com/2015/08/user-certificates-and-custom-profiles-with-freeipa-4-2/



Test Plan
=========

http://www.freeipa.org/page/V4/Certificate_Profiles/Test_Plan

Dependencies
============

-  Dogtag with LDAP profile replication enabled.
