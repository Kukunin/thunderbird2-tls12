## Thunderbird 2 with TLS1.2 support

### Repository

A couple of years ago, a bunch of vulnerabilities were found in SSL/TLS ciphers,
and many of servers nowadays (2019) dropped support for old vulnerable ciphers.
This made old HTTPS clients incompatible with modern internet.

Some people still use TB2 because of different reasons, and the most noticable
problem with it is lack of modern SSL/TLS ciphers support. So the client can't
access to huge part of servers that has dropped support of legacy ciphers.

For example, these modern ciphers are not supported:

* `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
* `TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256`
* `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
* `TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384`
* `TLS_RSA_WITH_AES_128_GCM_SHA256`
* `TLS_RSA_WITH_AES_256_GCM_SHA384`
* `TLS_RSA_WITH_AES_128_CBC_SHA256`

This repository adds support of modern ciphers to the old Thunderbird
`2.0.0.24` release. It was released in 2010 year.

### Motivation

Don't look at me in this way. It's a free-lance task, which I took to make.
Also it was useful for educational purposes.

### Research

We tried three different URLs:

- https://google.com - works with legacy ciphers
- https://developer.mozilla.org - works with some legacy ciphers
- https://services.addons.thunderbird.net - the most secure server - only new ciphers

You can check a server SSL capabilities in any online service, such as [this](https://www.immuniweb.com/ssl/?id=luzhIsoj).

For TB2, only google worked out of that URLs. Binary search showed,
that the least working version is `27.0b`. `26.0b` works supports 2 of 3 URLs,
while `25.0b` behaves the same as `2.0.0.24`.

So I took full NSS and NSPR libraries from `27.0b` and updated the management code.

### Testing

Thunderbird has JS Console, which allows arbitrary JS code to be evaluated.

I used the following snippet to check whether HTTPS works:

    (function(url) { const xhr = new XMLHttpRequest(); xhr.open("GET", url); xhr.onerror = function(err) { throw err; }; xhr.onload = function() { alert("good"); }; xhr.send(true); })("https://google.com")

or

    (function(url) { const xhr = new XMLHttpRequest(); xhr.open("GET", url); xhr.onerror = function(err) { throw err; }; xhr.onload = function() { alert("good"); }; xhr.send(true); })("https://services.addons.thunderbird.net")

### Building

Building such an old version is complicated task. The main problem is that
documentation is modern and it was hard to find the right old versions of tools.

Thunderbird 2 is based on Mozilla 1.8 branch. Here are useful links about it:

- [Windows Build Prerequisites (1.7 and 1.8 Branches)](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Windows_Build_Prerequisites_%281.7_and_1.8_Branches%29)
- [Windows build prerequisites using cygwin](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Windows_build_prerequisites_using_cygwin#Software_Requirements)

On Windows, it can be built with MSVC7 (Visual Studio 2003). 2003!!!.
MSVC8 (VS2005) is too modern for it, and it didn't work for me.

[MozillaBuild](https://wiki.mozilla.org/MozillaBuild) is available to provide
all dependencies on Windows. Versions 1.4 and 1.5 are known to work.
I couldn't find Windows SDK 7.0 so I used 7.1 without problems, with
only a little change in MozillaBuild startup scripts (`s/7.0/7.1/g`).

### Troubleshooting the build

* **Endless loop for: `Updating dependencies for: .deps/.all.pp`**
  Remove all `.deps` folders (`find . -name .deps | xargs rm -rf`) and try again

* **Undefined reference `__imp___wassert`**
  Edit source code to disable asserts, for example: `#define assert(x) (void)0`

* **Undefined reference `sqlite3_prepare` and other sqlite3-related functions**
  This is weird one. Despite they all are declared in `libsqlite3.so`, a linker
  doesn't pick them up for some reason. However, adding `ac_cv_visibility_pragma=no`
  to `mozconfig` fixes the issue. It might be related with `*.def` files.

* **Undefined reference font-related functions**
  For some reason, `configure` picked only `-lXft` for my environment.
  Edit `config/autoconf.mk` and append `-lfontconfig -lfreetype` to `-lXft`
  for `MOZ_XFT_LIBS`

* **Segmentation fault in nsIView::GetOffsetTo**
  Didn't solve, but worked around with early `return offset;` in the function in `view/src/nsView.cpp`.
