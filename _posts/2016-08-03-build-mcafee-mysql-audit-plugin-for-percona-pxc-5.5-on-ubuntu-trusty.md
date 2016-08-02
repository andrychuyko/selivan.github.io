---
layout: post
title:  "Building McAfee mysql audit plugin for Percona XtraDB Cluster 5.5 on Ubuntu 14.04.Trusty"
tags: [mysql,pxc,audit]
---
Percona XtraDB Cluster already includes audit plugin: [Percona Server 5.5 Audit Log Plugin](https://www.percona.com/doc/percona-server/5.5/management/audit_log_plugin.html). It is alternative implementation of MySQL Enterprise Audit Log Plugin by Oracle. Unfortunately, in 5.5 version it can not exclude some users from logging. For some use cases this faeture is crucial.

[McAfee mysql audit plugin](https://github.com/mcafee/mysql-audit) can do it for any mysql version: 5.5, 5.6, 5.7. There are no pre-compiled binaries for PXC 5.5, so we need to build it manualy, which is a little tricky.

Add Precona repository and install PXC packages, including packages with debug symbols:
```
wget https://repo.percona.com/apt/percona-release_0.1-3.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_0.1-3.$(lsb_release -sc)_all.deb
sudo apt-get update
sudo apt-get install percona-xtradb-cluster-server-5.5 percona-xtradb-cluster-5.5-dbg
```
Package `percona-xtradb-cluster-server-debug-5.5` looks suitable, but it's not what we need.

Install packages required to build PXC and plugin:
```
sudo apt-get build-dep percona-xtradb-cluster-server-5.5
sudo apt-get install git gdb autoconf
```
{% comment %}
To build mcafee-mysql-audit, we need automake-1.15. We can install it from [ondrej/autotools ppa](https://launchpad.net/~ondrej/+archive/ubuntu/autotools):
```
sudo add-apt-repository ppa:ondrej/autotools
sudo apt-get update
sudo apt-get install automake
```
{% endcomment %}

Get PXC sources:
```
apt-get source percona-xtradb-cluster-server-5.5
```

Get mcafee mysql-audit source for latest stable version(( See most recent tag for https://github.com/mcafee/mysql-audit )):
```
git clone https://github.com/mcafee/mysql-audit.git
cd mysql-audit
git checkout v1.0.9
```

Configure and build PXC sources. Some headers there are present in *.h.in form (m4 templates, used by autotools), and we need to get pure *.h. Also for 5.5 version we need static library libmysqlservices.a:
```
cpu_cores=$(cat /proc/cpuinfo | grep ^processor | wc -l)
cd ../percona-xtradb-cluster-5.5-5.5.41-25.11/
make -j $cpu_cores -f debian/rules configure && echo configure ok
make -j $cpu_cores -f debian/rules build && echo build ok
```
To run make faster, we can use `-j {number_of_processor_cores}`.

Here comes a tricky part: some of reuqired for plugin build headers are in `percona-xtradb-cluster-5.5-5.5.41-25.11`, and some in `percona-xtradb-cluster-5.5-5.5.41-25.11/builddir/`. May be there is more correct way to fix it, but here is the simplest:
```
cd ..
cp -rvn ./percona-xtradb-cluster-5.5-5.5.41-25.11/builddir/* ./percona-xtradb-cluster-5.5-5.5.41-25.11/
```

Now we can build the plugin:
```
cd mysql-audit/
autoreconf --force --install
./configure --with-mysql=$(readlink -f ..)/percona-xtradb-cluster-5.5-5.5.41-25.11 --with-mysql-libservices=$(readlink -f ..)/percona-xtradb-cluster-5.5-5.5.41-25.11/libservices/libmysqlservices.a && echo configure ok
make -j $cpu_cores
find . -name libaudit_plugin.so
sudo cp -v $(find . -name libaudit_plugin.so) /usr/lib/mysql/plugin/
```
`autoreconf` runs automatically `automake` and, if required, other autotools: `aclocal`, `autoheader`, `libtoolize`. `readlink -f` is used, because for some reason configure script doesn't like relative directory names.

Then we need to calculate offset, that plugin uses for accessing built-in MySQL data structures, that are not exposed though an API. Plugin has built-in offsets for some mysql versions, but not for PXC.
```
chmod a+x ./offset-extract/offset-extract.sh
./offset-extract/offset-extract.sh /usr/sbin/mysqld /usr/lib/debug/usr/sbin/mysqld

160802 22:46:32 [Warning] Using unique option prefix key_buffer instead of key_buffer_size is deprecated and will be removed in a future release. Please use the full name instead.
//offsets for: /usr/sbin/mysqld (5.5.41-37.0-55)
{"5.5.41-37.0-55","4aa67e7bbbde1b77a557fcbb7df995dc", 6576, 6624, 4112, 4624, 104, 2608, 96, 0, 32, 104, 136, 6728},

```
First script argument is mysqld executable and second - it's debugging symbols.

Let's try to run plugin. Add this to [mysqld] section of `my.ini`:
```
#
# * Audit plugin
#
plugin-load=AUDIT=libaudit_plugin.so
audit_validate_checksum=ON
audit_checksum=4aa67e7bbbde1b77a557fcbb7df995dc
audit_offsets=6576, 6624, 4112, 4624, 104, 2608, 96, 0, 32, 104, 136, 6728
audit_json_file=ON
audit_json_log_file=/var/log/mysql/mysql-audit.json
# Security - disable uninstall of audit plugin
audit_uninstall_plugin=OFF
```
Restart mysqld, check error.log and mysql-audit.json.
