\__NOTOC_\_

`#2642 <https://fedorahosted.org/freeipa/ticket/2642>`__ Add support for
enhanced SSHFP DNS records per RFC 6594

Overview
========

IPA supports automatic update of SSHFP DNS records for managed hosts in
the ``ipa-client-install`` script and in ``host-*`` commands. The
support is currently limited to the original SSHFP specification from
RFC 4255; SSHFP records generated by IPA contain SHA-1 fingerprints of
RSA and DSS host keys.

Recently, RFC 6594 was released. It extends the original SSHFP
specification with support for SHA-256 fingerprints and ECDSA host keys.

Add support for RFC 6594 SSHFP records to IPA, generate both SHA-1 and
SHA-256 fingerprints for RSA, DSS and ECDSA host keys.

.. _use_cases10l:

Use Cases
=========

Automatic generation of SSHFP DNS records on IPA client install:

| ``# ipa-client-install``
| ``Discovery was successful!``
| ``Hostname: host1.example.com``
| ``Realm: EXAMPLE.COM``
| ``DNS Domain: example.com``
| ``IPA Server: ipa.example.com``
| ``BaseDN: dc=example,dc=com``
| ``Continue to configure the system with these values? [no]: yes``
| ``User authorized to enroll computers: admin``
| ``Synchronizing time with KDC...``
| ``Password for admin@EXAMPLE.COM: ``
| ``Enrolled in IPA realm EXAMPLE.COM``
| ``Created /etc/ipa/default.conf``
| ``New SSSD config will be created``
| ``Configured /etc/sssd/sssd.conf``
| ``Configured /etc/krb5.conf for IPA realm EXAMPLE.COM``
| ``trying ``\ ```https://ipa.example.com/ipa/xml`` <https://ipa.example.com/ipa/xml>`__
| ``Hostname (host1.example.com) not found in DNS``
| ``DNS server record set to: host1.example.com -> 192.168.1.1``
| ``Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub``
| ``Adding SSH public key from /etc/ssh/ssh_host_dsa_key.pub``
| ``Forwarding 'host_mod' to server u'``\ ```https://ipa.example.com/ipa/xml`` <https://ipa.example.com/ipa/xml>`__\ ``'``
| ``SSSD enabled``
| ``Configured /etc/openldap/ldap.conf``
| ``NTP enabled``
| ``Configured /etc/ssh/ssh_config``
| ``Configured /etc/ssh/sshd_config``
| ``Client configuration complete.``

| ``$ dig host1.example.com SSHFP +short``
| ``2 2 0E04A7E09D037934492108ED5590612416BE736AD1BCAEAE1EA4148E 80C956E2``
| ``2 1 F2A1353FF919AD785B6BD42B588F6236D1F67459``
| ``1 2 3E475EEAF17975C36EE1413DDD659275FDD19C97C2C74A3651BA12F7 52E12A18``
| ``1 1 A308B1B02A8B43CB5192E26FA50280F752BB3A14``

Automatic generation of SSHFP DNS records when modifying a host:

| ``$ ipa host-mod host2.example.com --updatedns --sshpubkey='ssh-rsa ``\ ``' --sshpubkey='ssh-dss ``\ ``' --sshpubkey='ecdsa-sha2-nistp256 ``\ ``'``
| ``---------------------------------------------``
| ``Modified host "host2.example.com"``
| ``---------------------------------------------``
| ``  Host name: host2.example.com``
| ``  Principal name: host/host2.example.com@EXAMPLE.COM``
| ``  MAC address: 00:11:22:33:44:55``
| ``  SSH public key: ecdsa-sha2-nistp256 ``\ ``,``
| ``                  ssh-dss ``\ ``,``
| ``                  ssh-rsa ``
| ``  Keytab: True``
| ``  Managed by: host2.example.com``
| ``  SSH public key fingerprint: 6C:9F:07:51:63:36:32:8B:ED:CF:8C:4C:5F:F2:BF:AE (ecdsa-sha2-nistp256),``
| ``                              07:5D:0D:55:64:62:A3:FE:02:AE:FC:CD:F6:ED:E1:D9 (ssh-dss),``
| ``                              8C:C3:27:A8:40:9F:80:01:61:99:D2:25:55:A3:52:30 (ssh-rsa)``

| ``$ dig host2.example.com SSHFP +short``
| ``2 2 43FFD792089442F08892CA753059FD8B7FA939E990CE4687A3D1FB75 E0B8F6DE``
| ``2 1 4C2C50EDEAE6BC6107A37EAE7A05694C15CFEC53``
| ``3 1 B1D733A262E29B44A4D8A9FAF4B3B9E78302D1DB``
| ``1 2 E5382308CFD60DE4F0ACF3BCB0366314EECFC71030A28AAF75280041 5FDF81A8``
| ``3 2 545055E921E94128AF6BFE68E6E2804333628F7808B8EAE10E297B11 3270862F``
| ``1 1 DA7A6687AE4B2C242E12A67DACDC67D26E374AD5``

Design
======

Implement support for SHA-256 fingerprints and ECDSA keys in SSHFP
records in the ``ipapython.ssh`` module (add new method
``fingerprint_dns_sha256``).

Extend ``ipa-client-install`` and the ``host`` plugin to add all types
of SSHFP records to DNS.

Implementation
==============

N/A

.. _feature_managment:

Feature Managment
=================

N/A



Major configuration options and enablement
==========================================

N/A

Replication
===========

N/A



Updates and Upgrades
====================

N/A

Dependencies
============

N/A



External Impact
===============

N/A



RFE Author
==========

`Jan Cholasta <User:Jcholast>`__
