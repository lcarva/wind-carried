+++
title = 'Python Manylinux'
date = 2026-01-08T14:59:32-05:00
+++

I recently came across a part of the Python ecosystem that was completely new to me: `manylinux`.

If you're not familiar with Python packages, the gist is that, for the vast majority, they are
written purely in Python. Those are nice and easy to distribute. Just create a
[wheel](https://peps.python.org/pep-0427/), publish it to [PyPI.org](https://pypi.org/), and call
it a day. That wheel will work on any system that supports Python 3, which is, hopefully, most
users nowadays.

Great! But what about Python packages that contain non-Python code, e.g. C and Rust? Well... before
execution, this source code must first be compiled to binary code. The thing about binary code is
that it is not portable across different CPU architectures, different Operating Systems, and even
different Python 3 versions. The answer in the Python ecosystem is to create different wheels for
each combination you wish to support, e.g. Python 3.14 on Windows x86_64, Python 3.12 on Mac OSX 14
arm64, etc. The [platform compatibility
tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/#) specify
the various flavors. The combinations add up fast.

## But wait there is more

Some python packages also require using certain system libraries at run time, e.g.
[OpenSSL](https://www.openssl.org/). This is often dynamically linked in the wheel. When the Python
code needs to call such libraries, it loads the library that is currently available in the system.
This happens at *runtime*. Two systems with different versions of the system library will execute
different code. This is particularly challenging on Linux systems. Due to the many different
[flavors of Linux](https://en.wikipedia.org/wiki/Linux_distribution) out there, the `linux`
platform compatibility tag is discouraged. It is not sufficient to accommodate the different
nuances between different Linux distributions.

## Manylinux to the rescue!

The python community created a new platform tag, called
[manylinux](https://peps.python.org/pep-0513/), to increase the compatibility of Linux wheels. This
is achieved by changing the link to libraries from being dynamic to *static*. A manylinux wheel
contains a copy of the system libraries that were used at *build* time. If a python package, for
example, relies on OpenSSL, its manylinux wheel will contain a copy of OpenSSL.

The manylinux standard has evolved over time through [PEP 571](https://peps.python.org/pep-0571/),
[PEP 599](https://peps.python.org/pep-0599/), and [PEP 600](https://peps.python.org/pep-0600/) to
support newer Linux distributions and provide a more maintainable naming scheme. The standard
leverages glibc's superb [backwards
compatibility](https://developers.redhat.com/blog/2019/08/01/how-the-gnu-c-library-handles-backward-compatibility#)
properties.

## Wrapping up

What started as confusion about platform tags led me down a rabbit hole of Python packaging.
Manylinux represents years of community effort to make Python packages more reliable across the
diverse Linux landscape. The trade-off is clear: larger wheel sizes in exchange for the peace of
mind that packages like numpy, cryptography, and Pillow will "just work" on most Linux systems.
It's one of those invisible infrastructure pieces that most users never know exists but we all
benefit from it every day.
