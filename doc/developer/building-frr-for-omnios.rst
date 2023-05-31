OmniOS (OpenSolaris)
====================================================

OmniOS restrictions:
--------------------

-  MPLS is not supported on ``OmniOS`` or ``Solaris``. MPLS requires a
   Linux Kernel (4.5 or higher). LDP can be built, but may have limited
   use without MPLS

Enable IP & IPv6 forwarding
^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    routeadm -e ipv4-forwarding
    routeadm -e ipv6-forwarding

Install required packages
-------------------------

Add pkgsrc [https://pkgsrc.smartos.org/install-on-illumos/]:

::
    #
    # Copy and paste the lines below to install the latest bootstrap.
    #
    BOOTSTRAP_TAR="bootstrap-trunk-x86_64-20230313.tar.gz"
    BOOTSTRAP_SHA="9cbd71b08358d1acf66eb44faacc78bc2abc77df"
    
    # Download the bootstrap kit to the current directory.
    curl -O https://pkgsrc.smartos.org/packages/SmartOS/bootstrap/${BOOTSTRAP_TAR}
    
    # Verify the SHA1 checksum.
    [ "${BOOTSTRAP_SHA}" = "$(/bin/digest -a sha1 ${BOOTSTRAP_TAR})" ] || echo "ERROR: checksum failure"
    
    # Verify PGP signature.  This step is optional, and requires gpg.
    #curl -O https://pkgsrc.smartos.org/packages/SmartOS/bootstrap/${BOOTSTRAP_TAR}.asc
    #curl -sS https://pkgsrc.smartos.org/pgp/8254B861.asc | gpg2 --import
    #gpg2 --verify ${BOOTSTRAP_TAR}{.asc,}
    
    # Install bootstrap kit to /opt/local
    tar -zxpf ${BOOTSTRAP_TAR} -C /
    
    # Add to PATH/MANPATH.
    PATH=/opt/local/sbin:/opt/local/bin:$PATH
    MANPATH=/opt/local/man:$MANPATH
    
    pkgin -y update

    # Have below from `pkgin list` installed:
    bmake-20200524nb1    Portable (autoconf) version of NetBSD 'make' utility
    bootstrap-mk-files-20230225 *.mk files for the bootstrap bmake utility
    bsdinstall-20160108nb1 Portable version of the BSD install(1) program
    bzip2-1.0.8          Block-sorting file compressor
    cwrappers-20220403   pkgsrc compiler wrappers
    gcc12-libs-12.2.0    The GNU Compiler Collection (GCC) support shared libraries
    libarchive-3.4.3     Library to read/create different archive formats
    libyang-1.0.240      YANG data modeling language library
    mktools-20220614     Collection of pkgsrc mk infrastructure tools
    mozilla-rootcerts-1.0.20221204 Root CA certificates from the Mozilla Project
    ncurses-6.4          CRT screen handling and optimization package
    openssl-1.1.1t       Secure Socket Layer and cryptographic library
    pcre-8.45            Perl Compatible Regular Expressions library
    pkg_alternatives-1.7 Generic wrappers for programs with similar interfaces
    pkg_install-20211115 Package management and administration tools for pkgsrc
    pkg_install-info-4.5nb3 Standalone GNU info file installation utility
    pkgin-22.10.0nb1     Apt / yum like tool for managing pkgsrc binary packages
    pkgsrc-gnupg-keys-20221116 GnuPG keys for pkgsrc infrastructure
    readline-8.2nb1      GNU library that can recall and edit previous input
    sqlite3-3.41.1       SQL Database Engine in a C Library
    xz-5.4.1             General-purpose data compression software
    zlib-1.2.13          General purpose data compression library



Add packages:

::

    pkg install \
      developer/build/autoconf \
      developer/build/automake \
      developer/lexer/flex \
      developer/parser/bison \
      developer/object-file \
      developer/linker \
      developer/library/lint \
      developer/build/gnu-make \
      developer/gcc12 \
      library/idnkit \
      library/idnkit/header-idnkit \
      system/header \
      system/library/math/header-math \
      runtime/perl \
      library/python-3/pip-311 \
      ooce/library/json-c \
      git libtool pkg-config


Add libjson to Solaris equivalent of ld.so.conf

::

    crle -64 -l /opt/ooce/lib/amd64/ -u

Add pytest:

::
    python3 -m pip install "pytest<5"


Install Sphinx:::
    python3 -m pip install sphinx


::


Fix PATH for all users and non-interactive sessions. Edit
``/etc/default/login`` and add the following default PATH:

::
    PATH=/usr/gnu/bin/:/opt/local/sbin:/opt/local/bin:/usr/sbin:/sbin:/opt/ooce/sbin:/usr/bin:/opt/ooce/bin

Edit ``~/.profile`` and add the following default PATH:

::

    PATH=/usr/gnu/bin/:/opt/local/sbin:/opt/local/bin:/usr/sbin:/sbin:/opt/ooce/sbin:/usr/bin:/opt/ooce/bin

.. include:: building-libyang.rst

Get FRR, compile it and install it (from Git)
---------------------------------------------

**This assumes you want to build and install FRR from source and not
using any packages**

Add frr group and user
^^^^^^^^^^^^^^^^^^^^^^

::

    sudo groupadd -g 93 frr
    sudo groupadd -g 94 frrvty
    sudo useradd -g 93 -u 93 -G frrvty -c "FRR suite" \
        -d /nonexistent -s /bin/false frr

(You may prefer different options on configure statement. These are just
an example)

::

    git clone https://github.com/frrouting/frr.git frr
    cd frr
    ./bootstrap.sh
    export MAKE=gmake
    export LDFLAGS="-L/opt/ooce/lib/amd64"
    CPPFLAGS="-I/opt/local/include -I/opt/ooce/include"
    export PKG_CONFIG_PATH=/opt/ooce/lib/amd64/:/opt/local/lib/pkgconfig/
    ./configure \
        --sysconfdir=/etc/frr \
        --enable-exampledir=/usr/share/doc/frr/examples/ \
        --localstatedir=/var/run/frr \
        --sbindir=/usr/lib/frr \
        --enable-multipath=64 \
        --enable-user=frr \
        --enable-group=frr \
        --enable-vty-group=frrvty \
        --enable-configfile-mask=0640 \
        --enable-logfile-mask=0640 \
        --enable-fpm \
        --with-pkg-git-version \
        --with-pkg-extra-version=-MyOwnFRRVersion
    gmake

Create "classic" Solaris packages
^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    mkdir  -p /var/tmp/frr-7.5/etc/frr/
    gmake DESTDIR=/var/tmp/frr-7.5 install
    cp zebra/zebra.conf.sample bgpd/bgpd.conf.sample bgpd/bgpd.conf.sample2 ripd/ripd.conf.sample ripngd/ripngd.conf.sample ospfd/ospfd.conf.sample ospf6d/ospf6d.conf.sample /var/tmp/frr-7.5/etc/frr/
    cp solaris/str.h solaris/pqueue.h /var/tmp/frr-7.5/usr/local/include/frr/
    cp /usr/sadm/install/scripts/i.manifest solaris/
    cp /usr/sadm/install/scripts/r.manifest solaris7
    cd solaris/
    gmake DESTDIR=/var/tmp/frr-7.5 packages


Install FRR locally
^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    gmake check
    sudo gmake install

Enable IP & IPv6 forwarding
^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    routeadm -e ipv4-forwarding
    routeadm -e ipv6-forwarding
