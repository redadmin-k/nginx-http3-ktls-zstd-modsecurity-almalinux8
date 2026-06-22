# Nginx HTTP/3 KTLS zstd ModSecurity RPM for AlmaLinux 8

This repository provides an unofficial Nginx RPM packaging spec for AlmaLinux 8.

This build is based on `nginx-http3-ktls-zstd-Almalinux10` and rebuilt for AlmaLinux 8 with ModSecurity WAF integration.

## Features

This Nginx build includes:

```text
Nginx 1.30.3
OpenSSL 3.5.x
KTLS
HTTP/3
zstd module
ModSecurity-nginx connector
```

The purpose of this repository is to provide a verification build for Nginx WAF integration on AlmaLinux 8.

## Important Notice

This repository is unofficial.

It is not provided, maintained, endorsed, or supported by AlmaLinux OS Foundation, Nginx, OWASP, Trustwave, OpenSSL, or any upstream project.

Use this repository at your own risk.

No warranty is provided.
Please test carefully in a verification environment before using it in production.

## Requirement

Before installing or building this Nginx RPM, the ModSecurity RPM must be installed first.

ModSecurity RPM for AlmaLinux 8:

```text
https://github.com/redadmin-k/modsecurity-almalinux8
```

This Nginx package is built with the ModSecurity-nginx connector and links to `libmodsecurity.so.3`.

Expected ModSecurity library path:

```text
/opt/modsecurity/lib64/libmodsecurity.so.3
```

Example check:

```bash
ldd /usr/sbin/nginx | grep modsec
```

Expected result:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

## Build Dependency

The ModSecurity RPM must be available in the build environment.

For mock builds, add the ModSecurity RPM to a local repository and enable it in the mock configuration before building this Nginx RPM.

Example package:

```text
modsecurity-3.0.15-1.el8.x86_64.rpm
```

## Runtime Verification

After installation, confirm that Nginx and ModSecurity are installed.

```bash
rpm -qa | grep nginx
rpm -qa | grep modsec
ldd /usr/sbin/nginx | grep modsec
```

Example result:

```text
nginx-1.30.3-1.ymir_el8.ngx.x86_64
modsecurity-3.0.15-1.el8.x86_64
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

## ModSecurity Configuration

This repository provides the Nginx build with ModSecurity support.

ModSecurity rules and configuration should be tested separately before enabling blocking mode in production.

Recommended first step:

```apache
SecRuleEngine DetectionOnly
```

After sufficient verification, blocking mode can be enabled if appropriate.

```apache
SecRuleEngine On
```

## Target Environment

Tested target environment:

```text
AlmaLinux 8
Nginx 1.30.3
OpenSSL 3.5.x
ModSecurity v3
ModSecurity-nginx connector
```

CentOS 7 may require additional dependency RPMs and separate verification.

## License

Nginx, OpenSSL, ModSecurity, zstd, and related upstream components are distributed under their respective upstream licenses.

Packaging files in this repository are provided for RPM build and integration testing purposes.

Unless otherwise stated, packaging files in this repository are released under the Apache License 2.0.

