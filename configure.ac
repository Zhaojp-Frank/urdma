# configure.ac

# Userspace Software iWARP library for DPDK
#
# Authors: Patrick MacArthur <patrick@patrickmacarthur.net>
#
# Copyright (c) 2016, University of New Hampshire InterOperability Laboratory
#
# This software is available to you under a choice of one of two
# licenses.  You may choose to be licensed under the terms of the GNU
# General Public License (GPL) Version 2, available from the file
# COPYING in the main directory of this source tree, or the
# BSD license below:
#
#   Redistribution and use in source and binary forms, with or
#   without modification, are permitted provided that the following
#   conditions are met:
#
#   - Redistributions of source code must retain the above copyright notice,
#     this list of conditions and the following disclaimer.
#
#   - Redistributions in binary form must reproduce the above copyright
#     notice, this list of conditions and the following disclaimer in the
#     documentation and/or other materials provided with the distribution.
#
#   - Neither the name of IBM nor the names of its contributors may be
#     used to endorse or promote products derived from this software without
#     specific prior written permission.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
# EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS
# BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN
# ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
# CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

AC_PREREQ(2.60)
AC_INIT([urdma], [m4_esyscmd([sh config/version.sh])],
        [https://github.com/zrlio/urdma/issues],
	[urdma], [https://github.com/zrlio/urdma])
AC_CONFIG_AUX_DIR([config])
m4_ifdef([AM_PROG_AR], [AM_PROG_AR])
AM_INIT_AUTOMAKE([1.11 foreign no-define subdir-objects -Wall -Wno-override])
AM_SILENT_RULES([yes])
LT_PREREQ([2.2])
LT_INIT
AC_CONFIG_MACRO_DIR([config])
AC_CONFIG_SRCDIR([src/liburdma/verbs.c])
AC_CONFIG_HEADERS([config.h])

AC_ARG_VAR([RTE_SDK], [Location of DPDK SDK installation])
AC_ARG_VAR([RTE_TARGET], [DPDK target system and toolchain])
AC_ARG_VAR([MACHINE_CFLAGS], [target machine optimization C compiler flags])
AC_ARG_VAR([KERNELDIR], [kernel source directory])
AC_SUBST([KERNELDIR])
if test ${RTE_SDK}x = x; then
	AC_MSG_ERROR([urdma requires DPDK.  Set RTE_SDK to the DPDK install location])
fi
if test ${RTE_TARGET}x = x; then
	AC_MSG_ERROR([urdma requires DPDK.  Set RTE_TARGET to the DPDK compilation target])
fi

MACHINE_CFLAGS="-march=native"
AC_SUBST([MACHINE_CFLAGS])
DPDK_CPPFLAGS="-I${RTE_SDK}/${RTE_TARGET}/include -include rte_config.h"
AC_SUBST([DPDK_CPPFLAGS])
DPDK_LDFLAGS="-L${RTE_SDK}/${RTE_TARGET}/lib"
AC_SUBST([DPDK_LDFLAGS])

if test ${KERNELDIR}x = x; then
	KERNELDIR="/lib/modules/`uname -r`/build"
fi
if test ! -f ${KERNELDIR}/Makefile; then
	AC_MSG_ERROR([urdma requires a kernel source tree.  Install kernel-devel or equivalent])
fi

AC_ARG_ENABLE([werror], [AS_HELP_STRING([--enable-werror],
	      [Treat compiler warnings as errors])],
	      [case ${enable_werror} in
	       yes) EXTRA_CFLAGS="-Werror" ;;
	       no) ;;
	       *) AC_MSG_ERROR([Invalid value ${enable_werror} for --enable-werror]) ;;
	       esac])
AC_SUBST([EXTRA_CFLAGS])

AC_PROG_CC
CFLAGS="-std=gnu11 ${CFLAGS}"
AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#include <assert.h>
static_assert(1, "compiler supports static_assert");]])],
[], AC_MSG_ERROR([Compiler does not support static_assert]))
AC_PROG_INSTALL
AC_PROG_MAKE_SET
AM_PROG_CC_C_O
AC_PROG_MKDIR_P

AC_SEARCH_LIBS([verbs_register_driver], [ibverbs], [], [AC_MSG_ERROR([dnl
urdma requires libibverbs >= 1.2.0])])

AC_MSG_CHECKING([for ibverbs driver library extension])
dummy=if$$
cat <<IBV_VERSION > $dummy.c
#include <infiniband/driver.h>
IBV_DEVICE_LIBRARY_EXTENSION
IBV_VERSION
IBV_DEVICE_LIBRARY_EXTENSION=`$CC $CPPFLAGS -E $dummy.c 2> /dev/null | tail -1`
rm -f $dummy.c
if test $IBV_DEVICE_LIBRARY_EXTENSION = IBV_DEVICE_LIBRARY_EXTENSION; then
	AC_MSG_RESULT([not defined])
	AC_MSG_ERROR([libibverbs does not define IBV_DEVICE_LIBRARY_EXTENSION])
else
	AC_MSG_RESULT([$IBV_DEVICE_LIBRARY_EXTENSION])
fi
AC_SUBST([IBV_DEVICE_LIBRARY_EXTENSION])

AC_SEARCH_LIBS([rdma_create_id], [rdmacm], [], [AC_MSG_ERROR([dnl
urdma requires librdmacm])])
PKG_CHECK_MODULES([LIBNL3], [libnl-3.0 libnl-route-3.0])
AC_MSG_CHECKING([whether netlink/route/link.h includes linux/if.h])
old_CFLAGS="$CFLAGS"
old_LIBS="$LIBS"
CFLAGS="$CFLAGS $LIBNL3_CFLAGS"
LIBS="$LIBS $LIBNL3_LIBS"
AC_LINK_IFELSE([AC_LANG_PROGRAM([[#include <netlink/route/link.h>]],
	       [[int main(void) {
			int x = IFF_UP;
			return 0;
		}]])],
		[AC_MSG_RESULT([yes])]
		[AVOID_NET_IF_H=1],
		[AC_MSG_RESULT([no])]
		[AVOID_NET_IF_H=0])
AC_DEFINE_UNQUOTED([AVOID_NET_IF_H], [$AVOID_NET_IF_H],
		   [Define to 1 if netlink/route/link.h includes linux/if.h])
CFLAGS="$old_CFLAGS"
LIBS="$old_LIBS"

PKG_CHECK_MODULES([JSON_C], [json-c])

AC_HEADER_ASSERT

old_CFLAGS="${CFLAGS}"
old_CPPFLAGS="${CPPFLAGS}"
old_LDFLAGS="${LDFLAGS}"
CFLAGS="${CFLAGS} ${MACHINE_CFLAGS}"
CPPFLAGS="${CPPFLAGS} ${DPDK_CPPFLAGS}"
LDFLAGS="${CPPFLAGS} ${DPDK_LDFLAGS}"
AC_CHECK_HEADERS([rte_ethdev.h], [],
[AC_MSG_ERROR([urdma requires DPDK >= 2.2.0])])

DPDK_LIBS=
AC_SUBST([DPDK_LIBS])
for lib in dpdk intel_dpdk; do
	old_LIBS="${LIBS}"
	LIBS="-l${lib} ${LIBS}"
	AC_LINK_IFELSE([AC_LANG_PROGRAM([[#include <rte_eal.h>]],
		[[int main(int argc, char *argv[]) {
			rte_eal_init(argc, argv); return 0;
		}]])],
	[DPDK_LIBS=-l${lib}; break], [])
	LIBS=${old_LIBS}
done
if test ${DPDK_LIBS}x = x; then
	AC_MSG_ERROR([urdma requires DPDK >= 2.2.0])
fi
DPDK_LDFLAGS="${DPDK_LDFLAGS} -Wl,--whole-archive,${DPDK_LIBS},--no-whole-archive"

AC_LINK_IFELSE([AC_LANG_PROGRAM([[#include <rte_kni.h>]],
	       [[int main(int argc, char *argv[]) {
			rte_eal_init(argc, argv);
			rte_kni_init(1);
			return 0;
		}]])], [],
[AC_MSG_ERROR([urdma requires that DPDK be built with KNI support])])

CFLAGS="${old_CFLAGS}"
CPPFLAGS="${old_CPPFLAGS}"
LDFLAGS="${old_LDFLAGS}"

AC_CONFIG_FILES([Makefile src/kmod/Makefile])
AC_OUTPUT

AS_ECHO(["
-----------------------------------------------------------------------
${PACKAGE_NAME} Version ${PACKAGE_VERSION}

Using installation prefix: ${prefix}
Using libdir: ${libdir}
Using includedir: ${includedir}
Using bindir: ${bindir}
Using C compiler: ${CC} ${EXTRA_CFLAGS} ${CFLAGS} ${CPPFLAGS}

You can now run 'make' to build and 'make install' to install.
-----------------------------------------------------------------------
"])