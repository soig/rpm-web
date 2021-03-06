---
layout: default
title: rpm.org - Conditional Builds
---
# Conditional Builds

Rpmbuild supports conditional package builds with the command line switches --with and --without.

Here is an example to enable gnutls and disable openssl support:

```
$ rpmbuild -ba newpackage.spec --with gnutls --without openssl
```

## Enable --with/--without parameters
To use this feature in a spec file, add this to the beginning of the spec:

```
# add --with gnutls option, i.e. disable gnutls by default
%bcond_with gnutls
# add --without openssl option, i.e. enable openssl by default
%bcond_without openssl 
```

If you want to change whether or not an option is enabled by default, only change %bcond_with to %bcond_without or vice versa. The remainder of the spec file needs to stay the same.

## Check wether an option is enabled
To define BuildRequires depending on the commandline switch, you can use the %{with foo} macro:

```
%if %{with gnutls}
BuildRequires:  gnutls-devel
%endif
%if %{with openssl}
BuildRequires:  openssl-devel
%endif
```

## Pass it to %configure

To pass this option to configure or other scripts than understand a --with-foo or --without-foo parameter, you can use the %{?_with_foo} macro:

```
%configure \
        %{?_with_gnutls} \
        %{?_with_openssl}
```

## References
* [doc/manual/conditionalbuilds](https://github.com/rpm-software-management/rpm/blob/master/doc/manual/conditionalbuilds)
* [macros](https://github.com/rpm-software-management/rpm/blob/master/macros.in) 

