# Nginx with QUIC (HTTP/3), Zstd, KTLS, and ModSecurity on AlmaLinux 8

This repository provides an unofficial Nginx build for AlmaLinux 8 with:

* HTTP/3 (QUIC) via OpenSSL
* Zstd compression via `zstd-nginx-module`
* KTLS (Kernel TLS offload) for sendfile-based TLS I/O
* ModSecurity WAF integration via `ModSecurity-nginx`

This build is intended for verification and testing of modern Nginx features on AlmaLinux 8.

## Repository Layout

```text
RPMS/     - Built binary RPM packages
SRPM/     - Source RPM package
SPEC/     - Spec file used for building
logs/     - Build logs from mock
README.md - Documentation and verification notes
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

This build uses OpenSSL 3.5.7 for TLS, QUIC, HTTP/3, and KTLS support.

Check the OpenSSL version reported by Nginx:

```bash
nginx -V 2>&1 | grep -E 'built with OpenSSL|running with OpenSSL'
```

Expected output example:

```text
built with OpenSSL 3.5.7
```

## Features

### 1. HTTP/3 (QUIC)

This build enables HTTP/3 support with OpenSSL 3.5.7.

Verification:

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

### 2. Zstd Compression

This build includes `zstd-nginx-module` to support `Content-Encoding: zstd`.

Verification:

```bash
curl -sS -D- -o /dev/null -H 'Accept-Encoding: zstd' https://alma8.redadmin.org/
```

Expected output example:

```text
HTTP/1.1 200 OK
Content-Encoding: zstd
```

### 3. KTLS (Kernel TLS)

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
modprobe tls
```

To load it automatically at boot:

```bash
echo tls > /etc/modules-load.d/tls.conf
```

Then confirm:

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

### 4. ModSecurity

This build includes the ModSecurity-nginx connector.

The ModSecurity library is expected to be provided by the separate ModSecurity RPM.

Expected library path:

```text
/opt/modsecurity/lib64/libmodsecurity.so.3
```

Verify that Nginx resolves `libmodsecurity.so.3`:

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

Confirm that Nginx was built with the ModSecurity connector:

```bash
nginx -V 2>&1 | grep -i modsecurity
```

Expected output should include:

```text
--add-module=ModSecurity-nginx
```

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
1. Install Nginx and ModSecurity packages.
2. Configure OWASP CRS manually.
3. Start with SecRuleEngine DetectionOnly.
4. Review audit logs.
5. Add local exclusions for false positives.
6. Enable SecRuleEngine On after validation.
7. Keep CRS configuration and local exclusions under version control.
```

ModSecurity and CRS should be treated as one layer of defense, not as a replacement for application fixes, patching, access control, or rate limiting.

## License

nginx - BSD 2-Clause License
OpenSSL - Apache License 2.0
zstd-nginx-module - BSD 2-Clause License
ModSecurity - Apache License 2.0
ModSecurity-nginx - Apache License 2.0

Packaging files in this repository are provided for RPM build and integration testing purposes.

## Notice

This is an unofficial Nginx build for AlmaLinux 8.

This repository is not affiliated with, endorsed by, or supported by the AlmaLinux OS Foundation, Nginx, OpenSSL, OWASP, Trustwave, or any upstream project.

Use these RPM packages at your own risk.

