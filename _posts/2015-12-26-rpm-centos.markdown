---
layout: post
title:  "RPM differences across CentOS versions"
date:   2015-12-26 00:00:00
categories: rpm CentOS RHEL RedHat packaging linux signing GPG 
---

I've been working on building RPMs on CentOS 6, but which work on CentOS (and RedHat Enterprise) 5, 6 and 7. Every release of CentOS comes with a different release of rpm, and each release has it's own quirks. So far, I've been bitten by:

- RPM signing in CentOS 5 is pretty completely broken
- Directory attributes don't work in CentOS 5 and 6

RPM Signing
===

RPMs built on CentOS 6 or 7 won't install on CentOS 5. If you force installation with the `--no-signature` flag, the error will re-occur when listing or removing packages. This occasionally caused the `rpm` command to loop forever. The error is:

```
error: <your package>: Header V4 RSA/SHA1 signature: BAD, key ID <your key id>
error: <your package> cannot be installed
```

CentOS 5 doesn't support V4 signatures, which are the default in newer releases. There are also significant constraints on what types and sizes of keys can be used to sign packages on CentOS 5. Full instructions on how to configure keys and signing to support CentOS 5 are here: [Pitfalls with RPM and GPG](https://technosorcery.net/blog/2010/10/10/pitfalls-with-rpm-and-gpg/).

Directory Attributes
===

Both CentOS 5 and 6 fail to recognize directories with the `%attr` directive, which sets permissions for files included in the rpm. For example, given a stanza in the spec file like:

```
%files
%defattr(666, -, -, 666)
a
b
%attr(700, myself, myself) c
```

If `c` were a file, it would be installed with `700` permissions and `myself:myself` as the owner. But if `c` were a directory, the `%attr` stanza would be silently ignored and it would be installed with `666` permissions (from the `%defattr` from the package) and `root:root` (the RPM defaults) as the owner.

To make the permissions apply to the directory correctly, we have specify the `%defattr` and override it for every file (assuming `a` and `b` are files):

```
%files
%defattr(700, myself, myself, 700)
%attr(666, root, root) a
%attr(666, root, root) b
c
```

This will (as expected) create two files, `a` and `b`, which have permissions `666` and owner `root:root`, and also create directory `c` with permissions `700` and owner `myself:myself`.

The alternative is to use `chmod` in the post-install of the RPM. This is necessary if there are two directories which need different permissions, for example.
