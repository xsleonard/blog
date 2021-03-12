+++
title = "Distributing libcurl with SSL"
tags = ["software", "c", "c++", "libcurl", "SSL", "gnutls"]
date = "2014-01-14T00:00:00+00:00"
+++

Introduction
------------

This guide is intended for someone that needs to make https requests
from a closed-source C++ application to a limited set of hosts/sites. It
only covers distributing for Linux; Windows and OS X will be in a future
update.

In developing [Gnomescroll](https://gnomescroll.com), I needed a way to
authenticate users. It is a multiplayer-only game, so user accounts are
necessary to save progress. The original authentication system was
developed in Fall 2012. It relies on an https auth server (written in
Flask) and a shared-secret token based system for authenticating clients
with game servers.

* * * * *

Instructions
-----------------------------

### Libcurl source

[Download the libcurl source](https://github.com/bagder/curl)

### Compile libcurl

Libcurl uses both cmake and autoconf. We are going to use autoconf. The
[libcurl compilation guide](http://curl.haxx.se/docs/install.html) does
not mention this.

We are going to compile it as a shared library only, use GnuTLS for SSL,
and disable everything else except cookies. We are also going to bump
the library version to avoid a "no version information available" error
with ldd.

GnuTLS is LGPL, and everything we need it to link to is LGPL, so it is
supposedly ok for dynamic linking from a proprietary application (don't
take my word for it). OpenSSL is another option, but it is BSD 4-clause
licensed, which is incompatible with your application if it is GPL.

Libcurl has several other [options for SSL
libraries](http://curl.haxx.se/docs/ssl-compared.html).

Note that the various SSL libraries have slight differences in how
certificates are specified and located. I will only be covering
certificates with GnuTLS.

### Fetching the source and generating ./configure:

```sh
git clone https://github.com/bagder/curl.git
cd curl/
aclocal; autoheader; automake; autoconf
```

### Configuring for GnuTLS:


```sh
./configure --without-libidn --without-winidn --without-librtmp \
--without-libssh2 --without-libmetalink --without-ssl --with-gnutls \
--disable-ipv6 --disable-gopher --disable-smtp --disable-imap \
--disable-pop3 --disable-tftp --disable-telnet --disable-dict \
--disable-rtsp --disable-ldaps  --disable-ldap  --disable-file \
--disable-ftp --disable-static --disable-libcurl-option \
--disable-threaded-resolver --enable-versioned-symbols \
--enable-soname-bump --enable-cookies
```

### Configuring for OpenSSL:

```sh
./configure --without-libidn --without-winidn --without-librtmp \
--without-libssh2 --without-libmetalink --with-ssl --disable-ipv6 \
--disable-gopher --disable-smtp --disable-imap --disable-pop3  \
--disable-tftp --disable-telnet --disable-dict --disable-rtsp  \
--disable-ldaps  --disable-ldap  --disable-file --disable-ftp \
--disable-static --disable-libcurl-option --enable-cookies \
--disable-threaded-resolver --enable-versioned-symbols \
--enable-soname-bump
```

### Configuring without SSL

If you don't want SSL, replace any instances of `--with-ssl`,
`--with-gnutls`, and any other `--with-<ssl_lib>` lines you may have
added with `--without-ssl`.

### Other configuration options

`./configure --help` will show other available options if these
instructions are not exactly what you need.

### Cross compiling for 32-bit:

If you need to build 32-bit libcurl from a 64-bit system, add
`--host=i386-linux-gnu CFLAGS=-m32` to the configuration options.
i386-linux-gnu is the host name for 32-bit on ubuntu/mint multilib; you
may need to do `--host=i686-pc-linux-gnu` or similar on other systems.

### Compiling:

```sh
make
```


### Install for your project

We are not going to `make install`, so it does not clobber the libcurl
installed by your system. However, if you provided your own
non-conflicting --prefix to the ./configure script, it is not a problem.
Here, we copy the libs into the lib/ subfolder of our project.

```sh
cp -P ./lib/.libs/libcurl.so* /path/to/project/lib/
```

We also need to copy the headers, because libcurl generates header files
per-compilation, as opposed to having a universal header file like every
other package you've encountered:

```sh
mkdir -p /path/to/project/lib/include/curl
cp ./include/curl/*.h /path/to/project/lib/include/curl/
```

You don't have to put the headers in that path, but don't let them
clobber your system's headers with a rogue `make install`. This is
especially important if you are cross-compiling to 32-bit, because the
headers generated during 64-bit and 32-bit compilation are incompatible.
If you are cross-compiling, make sure to have a different lib/ and
include/ path for each arch in your project.

### Integrate with your build system

Update your build system to use the project- or arch-specific include
and link paths for libcurl (e.g. `-I./lib/include32 -L./lib/lin32`).

* * * * *

Distributing your SSL keys
---------------------------------------------------------

You may not wish to rely on the user to have the correct certs installed
on their system. If the https sites you need to navigate to are limited
and known ahead of time, you can collect and include them with your
application.

### Get the public keys for your site(s)

You will need to download the public keys for all of your sites, and
every key in the chain. To retrieve most of the chain, optionally do:

```sh
gnutls-cli --print-cert example.com < /dev/null > example.com.pem
```

In my case, this retrieves all the keys in the chain except for the root
key. I have no idea why so you may experience different issues with your
sites. If you have problems validating your certs later, you can get
them all manually.

`gnutls-cli` is provided by the package `gnutls-bin` in ubuntu/mint.

To get the final root key, or all of your keys if you skipped
`gnutls-cli`, export them manually from the browser. With Chromium, the
steps are:

1.  Navigate to [https://example.com](https://example.com)
2.  Click the lock icon next to the url in the omnibar.
3.  Click 'Connection' tab
4.  Click 'Certificate Information' link
5.  Click 'Details' tab
6.  Click the root authority (the top one in the hierarchy box, "Builtin
    Object Token")
7.  Click 'Export', save to cert1.pem
8.  Repeat steps 6 and 7 for each level of the hierarchy, if you did not
    get the certs with gnutls-cli, or if you have problems with those
    certs.

### Combine the public keys

Combine all the keys you fetched into one .pem file.

```sh
cat cert*.pem > example.com.pem
rm cert*.pem
```

Copy them into your project's data or assets folder:

```sh
mkdir -p /path/to/project/assets/certs/
mv example.com.pem /path/to/project/assets/certs/
```

This method is only known to work with GnuTLS; OpenSSL handles certs
slightly differently. If you are using OpenSSL, you may need to do
`c_rehash /path/to/project/assets/certs/` after you copy the .pem file
in.

* * * * *

Making the SSL request from your application
---------------------------------------------------------------------------------------------

[Example C code for making an SSL request with the certs you
created](https://gist.github.com/xsleonard/6046486)

That example includes the cookie handling example from the libcurl
website, because you'll likely need it for your authentication system.
Instructions for compiling it are near the top of the file.

After compiling it, test it:

```sh
./curltest https://example.com ./path/to/certs/example.com.pem
```

You should see the html content displayed in the terminal.

Remaining tasks
-----------------------------------

This still isn't ready for full distribution. We have to:

-   Include the gnutls library and its dependencies in the lib folder.
    Possibly compile from source to restrict its dependencies.
-   Compile libcurl for windows and OS X for a cross-platform release

Also, to finish the rest of the authentication system, you'll need to
have:

-   A web server to authenticate with
-   A way for users to enter credentials into your app
-   A system for authenticating with a 3rd party, e.g. your game server
