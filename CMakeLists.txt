# Copyright (C) 2007-2012 LuaDist.
# Submited by David Manura
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

#~WARNING: The was derived from the Makefile and some parts
#   remain unimplemented:
#   gcc-2, x86-64, CFLAGS_P, bcheck.o, arm, TCC_UCLIBC,
#   CONFIG_CROSS
# NOTE: tcc itself is broken on Cygwin.

project ( tcc C )
cmake_minimum_required ( VERSION 2.8 )
include ( cmake/dist.cmake )

#~TODO: autodetect?
set ( ARCH i386 )

# Build config.h
configure_file ( ${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h )

set ( CORE_FILES tcc.c libtcc.c tccpp.c tccgen.c tccelf.c tccasm.c tcc.h config.h 
  libtcc.h tcctok.h )
set ( I386_FILES ${CORE_FILES} i386-gen.c i386-asm.c i386-asm.h )
set ( WIN32_FILES ${CORE_FILES} i386-gen.c i386-asm.c i386-asm.h tccpe.c )
set ( X86_64_FILES ${CORE_FILES} x86_64-gen.c )
set ( ARM_FILES ${CORE_FILES} arm-gen.c )
set ( C67_FILES ${CORE_FILES} c67-gen.c tcccoff.c )

# Set C compiler flags.
include ( CheckCCompilerFlag )
check_c_compiler_flag ( -fno-strict-aliasing, HAVE_NOSTRICTALIASING )
if ( HAVE_NOSTRICTALIASING )
  add_definitions ( -fno-strict-aliasing )
endif ( HAVE_NOSTRICTALIASING )

# Set target and sources
if ( WIN32 AND NOT CYGWIN )
  set ( NATIVE_TARGET -DTCC_TARGET_PE )
  set ( NATIVE_FILES ${WIN32_FILES} )
else ( )
  if ( ${ARCH} STREQUAL i386 )
    set ( NATIVE_TARGET -DTCC_TARGET_I386 )
    set ( NATIVE_FILES ${I386_FILES} )
  elseif ( ${ARCH} STREQUALS arm )
  set ( NATIVE_TARGET -DTCC_TARGET_ARM )
  #NOT-IMPL:
  # NATIVE_TARGET+=$(if $(wildcard /lib/ld-linux.so.3),-DTCC_ARM_EABI)
  # NATIVE_TARGET+=$(if $(shell grep -l "^Features.* \(vfp\|iwmmxt\) "
  #                /proc/cpuinfo),-DTCC_ARM_VFP)
  message ( FATAL_ERROR "TODO - not implemented" )
  elseif ( ${ARCH} STREQUALS x86-64 )
  set ( NATIVE_TARGET -DTCC_X86_64 )
  endif ( ${ARCH} STREQUAL i386 )
  # (CMAKE_SIZEOF_VOID_P MATCHES "8")
endif ( WIN32 AND NOT CYGWIN )

# Set C flags.
add_definitions ( ${NATIVE_TARGET} )

# Set libraries.
if ( NOT WIN32 )
  link_libraries ( -ldl )
endif ( NOT WIN32 )

# Set libtcc1 sources and tcc arguments
set ( LIBTCC1_SRC lib/libtcc1.c )
set ( TCC ${CMAKE_CURRENT_BINARY_DIR}/tcc )
set ( TCC_ARGS )
if ( WIN32 AND NOT CYGWIN )
  set ( LIBTCC1_SRC ${LIBTCC1_SRC} win32/lib/crt1.c win32/lib/wincrt1.c win32/lib/dllcrt1.c 
    win32/lib/dllmain.c win32/lib/chkstk.S )
  set ( LIBTCC1_OBJS ${LIBTCC1_OBJS} crt1.o wincrt1.o dllcrt1.o dllmain.o chkstk.o )
  set ( TCC_ARGS -Bwin32 -DTCC_TARGET_PE )
endif ( WIN32 AND NOT CYGWIN )
if ( ARCH STREQUAL i386 )
  set ( LIBTCC1_SRC ${LIBTCC1_SRC} lib/alloca86.S lib/alloca86-bt.S )
  set ( LIBTCC1_OBJS ${LIBTCC1_OBJS} alloca86.o alloca86-bt.o )
endif ( ARCH STREQUAL i386 )

# make 'include "config.h"' and 'include "../config.h"' work
file ( MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/child-dir )
include_directories ( ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/child-dir )

# TCC executable.
add_executable ( tcc tcc.c )

# libtcc
add_library ( libtcc STATIC libtcc.c )

# Extra utility executables.
if ( WIN32 AND NOT CYGWIN )
  add_executable ( tiny_impdef win32/tools/tiny_impdef.c )
endif ( WIN32 AND NOT CYGWIN )
add_executable ( tiny_libmaker win32/tools/tiny_libmaker.c )

set ( TCC_INCLUDE -I${CMAKE_CURRENT_BINARY_DIR}/child-dir )
# make include "../config.h" work
if ( WIN32 AND NOT CYGWIN )
  set ( TCC_INCLUDE ${TCC_INCLUDE} -I${CMAKE_CURRENT_SOURCE_DIR}/win32/include -I${CMAKE_CURRENT_SOURCE_DIR}/win32/include/winapi )
endif ( WIN32 AND NOT CYGWIN )

# libtcc1 sources built with tcc.
foreach ( src ${LIBTCC1_SRC} )
get_filename_component ( srcname ${src} NAME_WE )
get_filename_component ( srcext ${src} EXT )
if ( ${srcext} STREQUAL .c )
  set ( TCC_EXTRA_FLAGS -O2 -Wall )
else ( )
  # .S
  set ( TCC_EXTRA_FLAGS )
endif ( ${srcext} STREQUAL .c )
add_custom_command ( OUTPUT ${srcname}.o DEPENDS tcc ${src} COMMAND ${TCC} ${TCC_EXTRA_FLAGS} 
  ${TCC_INCLUDE} -c ${CMAKE_CURRENT_SOURCE_DIR}/${src} )
endforeach ( src )

# Link libtcc1
add_custom_command ( OUTPUT libtcc1.a DEPENDS tiny_libmaker ${LIBTCC1_OBJS} COMMAND 
  ${CMAKE_CURRENT_BINARY_DIR}/tiny_libmaker rcs libtcc1.a ${LIBTCC1_OBJS} )
add_custom_target ( libtcc1 ALL DEPENDS libtcc1.a )

# Install files.
install_executable ( tcc )
install_library ( libtcc )

install ( FILES ${CMAKE_CURRENT_BINARY_DIR}/libtcc1.a DESTINATION ${INSTALL_LIB}/tcc COMPONENT Library )

install_header ( libtcc.h )
install_doc ( tcc-doc.html )
install_header ( include/stdarg.h include/stddef.h include/stdbool.h include/float.h 
  include/varargs.h include/tcclib.h INTO tcc/include )
install_data ( README COPYING )
if ( WIN32 )
  install_header ( win32/include/assert.h win32/include/conio.h win32/include/ctype.h 
    win32/include/dir.h win32/include/direct.h win32/include/dirent.h win32/include/dos.h 
    win32/include/errno.h win32/include/excpt.h win32/include/fcntl.h win32/include/fenv.h 
    win32/include/float.h win32/include/inttypes.h win32/include/io.h win32/include/limits.h 
    win32/include/locale.h win32/include/malloc.h win32/include/math.h win32/include/mem.h 
    win32/include/memory.h win32/include/process.h win32/include/setjmp.h win32/include/share.h 
    win32/include/signal.h win32/include/stdarg.h win32/include/stdbool.h win32/include/stddef.h 
    win32/include/stdint.h win32/include/stdio.h win32/include/stdlib.h win32/include/string.h 
    win32/include/tchar.h win32/include/time.h win32/include/unistd.h win32/include/values.h 
    win32/include/varargs.h win32/include/wchar.h win32/include/wctype.h win32/include/_mingw.h 
    INTO tcc/include )
  install_header ( win32/include/sys/fcntl.h win32/include/sys/file.h win32/include/sys/locking.h 
    win32/include/sys/stat.h win32/include/sys/time.h win32/include/sys/timeb.h win32/include/sys/types.h 
    win32/include/sys/unistd.h win32/include/sys/utime.h INTO tcc/include/sys )
  install_header ( win32/include/winapi/basetsd.h win32/include/winapi/basetyps.h 
    win32/include/winapi/winbase.h win32/include/winapi/wincon.h win32/include/winapi/windef.h 
    win32/include/winapi/windows.h win32/include/winapi/winerror.h win32/include/winapi/wingdi.h 
    win32/include/winapi/winnetwk.h win32/include/winapi/winnls.h win32/include/winapi/winnt.h 
    win32/include/winapi/winreg.h win32/include/winapi/winsvc.h win32/include/winapi/winuser.h 
    win32/include/winapi/winver.h INTO tcc/include/winapi )
endif ( )
