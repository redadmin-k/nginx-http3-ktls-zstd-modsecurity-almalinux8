# Nginx with HTTP/3, KTLS, zstd, and ModSecurity for AlmaLinux 8

This repository provides an unofficial Nginx RPM build for AlmaLinux 8.

This build includes HTTP/3, OpenSSL 3.5.7, KTLS, zstd compression, and ModSecurity WAF integration.

## Features

- Nginx 1.30.3
- OpenSSL 3.5.7
- HTTP/3 / QUIC
- KTLS
- zstd compression via `zstd-nginx-module`
- ModSecurity support via `ModSecurity-nginx`

## Repository Layout

```text
RPMS/      - Built binary RPM packages
SRPM/      - Source RPM package
SPEC/      - Spec file used for building
logs/      - Build logs from mock
README.md  - Documentation and verification notes
```

## Important Notice

This repository is unofficial.

It is not provided, maintained, endorsed, or supported by the AlmaLinux OS Foundation, Nginx, OWASP, Trustwave, OpenSSL, zstd, ModSecurity, or any upstream project.

Use these RPM packages at your own risk.

No warranty is provided. Please test carefully in a verification environment before using them in production.

## Requirement

Before installing this Nginx RPM, install the ModSecurity RPM first.

ModSecurity RPM for AlmaLinux 8:

```text
https://github.com/redadmin-k/modsecurity-almalinux8
```

Install the ModSecurity RPM first:

```bash
git clone https://github.com/redadmin-k/modsecurity-almalinux8.git
cd modsecurity-almalinux8
sudo dnf localinstall $(find RPMS -type f -name '*.rpm' | sort)
```

This Nginx build expects `libmodsecurity.so.3` to be available under:

```text
/opt/modsecurity/lib64/libmodsecurity.so.3
```

Confirm that ModSecurity is available:

```bash
ldconfig -p | grep modsecurity
```

Expected result example:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

## Installation

Install this Nginx RPM after installing the ModSecurity RPM.

```bash
sudo dnf localinstall $(find RPMS -type f -name '*.rpm' | sort)
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

## Build Environment

```text
OS: AlmaLinux 8 x86_64
Build Tool: mock
Mock Config: alma+epel-8-x86_64
Nginx Version: 1.30.3
OpenSSL Version: 3.5.7
ModSecurity Version: 3.0.15
```

Additional modules:

```text
ModSecurity-nginx
zstd-nginx-module
```

## Build Details

This build uses OpenSSL 3.5.7 for TLS, HTTP/3, QUIC, and KTLS support.

Check the OpenSSL version reported by Nginx:

```bash
nginx -V 2>&1 | grep -E 'built with OpenSSL|running with OpenSSL'
```

Expected output example:

```text
built with OpenSSL 3.5.7
```

Confirm that Nginx was built with the ModSecurity connector:

```bash
nginx -V 2>&1 | grep -i modsecurity
```

Expected output should include:

```text
--add-module=ModSecurity-nginx
```

Confirm that Nginx resolves `libmodsecurity.so.3`:

```bash
ldd /usr/sbin/nginx | grep modsecurity
```

Expected output:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

The Nginx binary should not contain an RPATH or RUNPATH for `/opt/modsecurity/lib64`.

Check:

```bash
readelf -d /usr/sbin/nginx | grep -E 'RPATH|RUNPATH'
```

Expected result:

```text
# no output
```

## HTTP/3 Verification

HTTP/3 can be verified with a client that supports HTTP/3.

Example:

```bash
curl -I --http3 https://alma8.redadmin.org/
```

Expected output example:

```text
HTTP/3 200
server: nginx/1.30.3
alt-svc: h3=":443"; ma=86400
```

Nginx access logs may also show HTTP/3 requests:

```text
"GET / HTTP/3" 200
```

## zstd Verification

zstd compression can be verified with:

```bash
curl -sS -D- -o /dev/null -H 'Accept-Encoding: zstd' https://alma8.redadmin.org/
```

Expected output example:

```text
HTTP/1.1 200 OK
Content-Encoding: zstd
```

## KTLS Verification

Linux Kernel TLS can be used for sendfile-based TLS data transfer when supported by the kernel, OpenSSL, cipher suite, and Nginx configuration.

This build is compiled with OpenSSL KTLS support:

```text
--with-openssl-opt=enable-ktls
```

To use KTLS, the Linux kernel TLS module may need to be loaded.

Check whether the module is loaded:

```bash
lsmod | grep '^tls'
```

Load it manually:

```bash
sudo modprobe tls
```

To load it automatically at boot:

```bash
echo tls | sudo tee /etc/modules-load.d/tls.conf
```

Confirm again:

```bash
lsmod | grep '^tls'
```

If KTLS is built into the kernel instead of provided as a module, `lsmod` may not show `tls`.

Nginx configuration may also require enabling KTLS through OpenSSL configuration commands, depending on the environment:

```nginx
ssl_conf_command Options KTLS;
```

A possible runtime indication is `sendfile()` activity from an Nginx worker process:

```text
sendfile(...)
```

KTLS behavior depends on kernel support, OpenSSL support, TLS version, cipher suite, and Nginx configuration.

## OWASP Core Rule Set

OWASP Core Rule Set is not bundled in this package.

CRS should be installed or placed manually.

A typical ModSecurity include layout is:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

For initial operation, OWASP CRS 3.3.x is recommended because it is stable and easier to validate with ModSecurity v3.

CRS 4.x may work, but it should be validated separately before production use.

## ModSecurity Configuration Example

Example Nginx server configuration:

```nginx
server {
    listen 80;
    server_name alma8.redadmin.org;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

Example `/etc/nginx/modsec/main.conf`:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

For initial production testing, start with detection-only mode:

```apache
SecRuleEngine DetectionOnly
```

After reviewing audit logs and tuning false positives, blocking mode can be enabled:

```apache
SecRuleEngine On
```

## Simple WAF Test

After CRS is installed and blocking mode is enabled, a simple SQL injection test can be used for verification:

```bash
curl -i -H 'Host: alma8.redadmin.org' \
  'http://127.0.0.1/?id=1%20UNION%20SELECT%201,2,3'
```

Expected result when blocking is active:

```text
HTTP/1.1 403 Forbidden
```

If the request is detected but still returns `200`, check CRS blocking evaluation and local exclusions, especially rule `949110`.

## Production Notes

Do not enable blocking mode directly in production without verification.

Recommended rollout:

```text
1. Install the ModSecurity RPM.
2. Install this Nginx RPM.
3. Configure OWASP CRS manually.
4. Start with SecRuleEngine DetectionOnly.
5. Review audit logs.
6. Add local exclusions for false positives.
7. Enable SecRuleEngine On after validation.
8. Keep CRS configuration and local exclusions under version control.
```

ModSecurity and CRS should be treated as one layer of defense, not as a replacement for application fixes, patching, access control, rate limiting, or regular security updates.

## License

nginx - BSD 2-Clause License  
OpenSSL - Apache License 2.0  
zstd-nginx-module - BSD 2-Clause License  
ModSecurity - Apache License 2.0  
ModSecurity-nginx - Apache License 2.0  

Packaging files in this repository are provided for RPM build and integration testing purposes.

## Notice

This is an unofficial Nginx build for AlmaLinux 8.

This repository is not affiliated with, endorsed by, or supported by the AlmaLinux OS Foundation, Nginx, OpenSSL, OWASP, Trustwave, ModSecurity, or any upstream project.

Use these RPM packages at your own risk.
