+++
title = 'Using Host Certificates with Podman'
date = 2025-04-14T16:44:47-04:00
+++

Let's say you want to launch a container locally with [podman](https://podman.io/). Now, let's say
this container needs access to resources within your company's internal network which use a custom
[root CA](https://en.wikipedia.org/wiki/Root_certificate) (Certificate Authority). You will
certainly face certificate verification errors.

This can be frustrating because, after all, you have already trusted that root CA for your host.
This post is about extending that trust to containers launched by podman.

## Failed Attempt

Your first attempt might be to simply mount your local CA bundle when launching the container, e.g.

```bash
podman run -it \
 -v /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem \
 ...
```

Although the container may start successfully, outgoing connections will fail with certificate
verification errors. If you shell into the container and list the contents of
`/etc/pki/ca-trust/extracted/pem`, you will see something like this:

```bash
$ ls -la /etc/pki/ca-trust/extracted/pem/
ls: cannot access '/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem': Permission denied
total 660
drwxr-xr-x. 3 root root    154 Sep 18  2024 .
drwxr-xr-x. 6 root root     48 Sep 18  2024 ..
dr-xr-xr-x. 2 root root  15128 Sep 18  2024 directory-hash
-r--r--r--. 1 root root 165521 Sep 18  2024 email-ca-bundle.pem
-r--r--r--. 1 root root 502506 Sep 18  2024 objsign-ca-bundle.pem
-rw-r--r--. 1 root root    898 Aug 19  2024 README
-?????????? ? ?    ?         ?            ? tls-ca-bundle.pem
```

[SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) is preventing access to the
`tls-ca-bundle.pem` file. You may be tempted to simply disable SELinux, but consider reading
instead of taking this nuclear option.

## Success

The `container_read_certs` SELinux boolean was
[added](https://github.com/containers/container-selinux/pull/255) for the specific purpose of
allowing using the host's certificates in a container. This is likely disabled by default on your
host, but it is simple to enable it:

```bash
$ sudo getsebool container_read_certs # returns "off"
$ sudo setsebool -P container_read_certs 1
$ sudo getsebool container_read_certs # returns "on"
```

Let's verify on a new container:

```bash
$ podman run -it \
 -v /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem \
 ...

$ ls -la /etc/pki/ca-trust/extracted/pem/
total 900
drwxr-xr-x. 3 root   root      154 Sep 18  2024 .
drwxr-xr-x. 6 root   root       48 Sep 18  2024 ..
dr-xr-xr-x. 2 root   root    15128 Sep 18  2024 directory-hash
-r--r--r--. 1 root   root   165521 Sep 18  2024 email-ca-bundle.pem
-r--r--r--. 1 root   root   502506 Sep 18  2024 objsign-ca-bundle.pem
-rw-r--r--. 1 root   root      898 Aug 19  2024 README
-r--r--r--. 1 nobody nobody 243864 Feb 28 18:10 tls-ca-bundle.pem
```

Yay! Outgoing requests originating from the container will now trust the same root CAs trusted by
the host.

## Bundle Paths

The path to the certificate authority bundle varies between different distributions. You must know
the path used by both the host's Operating System and the container image. The path used above,
`/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem`, works in my case because my host is running
Fedora and the image being executed is based on RHEL. Both use the same path. If I launch a
container based on the alpine image, for example, a different volume mount target must be used:

```bash
podman run \
 -v /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/etc/ssl/certs/ca-certificates.crt \
 ...
```

Although not comprehensive, [this](https://go.dev/src/crypto/x509/root_linux.go) is a helpful
resource in determining the certificate authority bundle location for different distributions.

## Bonus

Some tools and libraries do not honor the system's certificate authority bundles. Instead, they
provide their own bundle. This is the case for python's
[requests](https://requests.readthedocs.io/en/latest/user/advanced/#ssl-cert-verification) and
[httpx](https://www.python-httpx.org/advanced/ssl/#configuring-client-instances) packages. A useful
package to consider is [truststore](https://github.com/sethmlarson/truststore) which has a helper
function for making those libraries use the host's CA bundle:

```python
import truststore
truststore.inject_into_ssl()
```

