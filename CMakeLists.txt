# Copyright (C) 2007-2009 LuaDist.
# Submited by David Manura
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

#~WARNING: The was derived from the Makefile and some parts
#   remain unimplemented:
#   gcc-2, x86-64, CFLAGS_P, bcheck.o, arm, TCC_UCLIBC,
#   CONFIG_CROSS
# NOTE: tcc itself is broken on Cygwin.


PROJECT(tcc C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
INCLUDE(dist.cmake)

#~TODO: autodetect?
SET(ARCH i386)

# Build config.h
CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/config.h)

SET(CORE_FILES tcc.c libtcc.c tccpp.c tccgen.c tccelf.c tccasm.c
               tcc.h config.h libtcc.h tcctok.h)
SET(I386_FILES   ${CORE_FILES} i386-gen.c i386-asm.c i386-asm.h)
SET(WIN32_FILES  ${CORE_FILES} i386-gen.c i386-asm.c i386-asm.h tccpe.c)
SET(X86_64_FILES ${CORE_FILES} x86_64-gen.c)
SET(ARM_FILES    ${CORE_FILES} arm-gen.c)
SET(C67_FILES    ${CORE_FILES} c67-gen.c tcccoff.c)

# Set C compiler flags.
INCLUDE(CheckCCompilerFlag)
CHECK_C_COMPILER_FLAG(-fno-strict-aliasing, HAVE_NOSTRICTALIASING)
IF(HAVE_NOSTRICTALIASING)
  ADD_DEFINITIONS(-fno-strict-aliasing)
ENDIF(HAVE_NOSTRICTALIASING)


# Set target and sources
IF(WIN32 AND NOT CYGWIN)
  SET(NATIVE_TARGET -DTCC_TARGET_PE)
  SET(NATIVE_FILES ${WIN32_FILES})
ELSE()
  IF(${ARCH} STREQUAL i386)
    SET(NATIVE_TARGET -DTCC_TARGET_I386)
    SET(NATIVE_FILES ${I386_FILES})
  ELSEIF(${ARCH} STREQUALS arm)
    SET(NATIVE_TARGET -DTCC_TARGET_ARM)
    #NOT-IMPL:
    # NATIVE_TARGET+=$(if $(wildcard /lib/ld-linux.so.3),-DTCC_ARM_EABI)
    # NATIVE_TARGET+=$(if $(shell grep -l "^Features.* \(vfp\|iwmmxt\) "
    #                /proc/cpuinfo),-DTCC_ARM_VFP)
    MESSAGE(FATAL_ERROR "TODO - not implemented")
  ELSEIF(${ARCH} STREQUALS x86-64)
    SET(NATIVE_TARGET -DTCC_X86_64)
  ENDIF(${ARCH} STREQUAL i386)
  # (CMAKE_SIZEOF_VOID_P MATCHES "8")
ENDIF(WIN32 AND NOT CYGWIN)

# Set C flags.
ADD_DEFINITIONS(${NATIVE_TARGET})


# Set libraries.
IF(NOT WIN32)
  LINK_LIBRARIES(-ldl)
ENDIF(NOT WIN32)



# Set libtcc1 sources and tcc arguments
SET(LIBTCC1_SRC lib/libtcc1.c)
SET(TCC ${CMAKE_CURRENT_BINARY_DIR}/tcc)
SET(TCC_ARGS)
IF(WIN32 AND NOT CYGWIN)
  SET(LIBTCC1_SRC  ${LIBTCC1_SRC}
      win32/lib/crt1.c win32/lib/wincrt1.c
      win32/lib/dllcrt1.c win32/lib/dllmain.c win32/lib/chkstk.S)
  SET(LIBTCC1_OBJS ${LIBTCC1_OBJS}
      crt1.o wincrt1.o dllcrt1.o dllmain.o chkstk.o)
  SET(TCC_ARGS -Bwin32 -DTCC_TARGET_PE)
ENDIF(WIN32 AND NOT CYGWIN)
IF(ARCH STREQUAL i386)
  SET(LIBTCC1_SRC  ${LIBTCC1_SRC}  lib/alloca86.S lib/alloca86-bt.S)
  SET(LIBTCC1_OBJS ${LIBTCC1_OBJS} alloca86.o alloca86-bt.o)
ENDIF(ARCH STREQUAL i386)


# make 'include "config.h"' and 'include "../config.h"' work
FILE(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/child-dir)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}
                    ${CMAKE_CURRENT_BINARY_DIR}/child-dir)

# TCC executable.
ADD_EXECUTABLE(tcc tcc.c)

# libtcc
ADD_LIBRARY(libtcc STATIC libtcc.c)

# Extra utility executables.
IF(WIN32 AND NOT CYGWIN)
  ADD_EXECUTABLE(tiny_impdef win32/tools/tiny_impdef.c)
ENDIF(WIN32 AND NOT CYGWIN)
ADD_EXECUTABLE(tiny_libmaker win32/tools/tiny_libmaker.c)



SET(TCC_INCLUDE -I${CMAKE_CURRENT_BINARY_DIR}/child-dir) # make include "../config.h" work
IF(WIN32 AND NOT CYGWIN)
  SET(TCC_INCLUDE ${TCC_INCLUDE} -I${CMAKE_CURRENT_SOURCE_DIR}/win32/include
                  -I${CMAKE_CURRENT_SOURCE_DIR}/win32/include/winapi)
ENDIF(WIN32 AND NOT CYGWIN)


# libtcc1 sources built with tcc.
FOREACH(src ${LIBTCC1_SRC})
  GET_FILENAME_COMPONENT(srcname ${src} NAME_WE)
  GET_FILENAME_COMPONENT(srcext  ${src} EXT)
  IF(${srcext} STREQUAL .c)
    SET(TCC_EXTRA_FLAGS -O2 -Wall)
  ELSE() # .S
    SET(TCC_EXTRA_FLAGS)
  ENDIF(${srcext} STREQUAL .c)
  ADD_CUSTOM_COMMAND(OUTPUT ${srcname}.o DEPENDS tcc ${src} COMMAND
      ${TCC} ${TCC_EXTRA_FLAGS} ${TCC_INCLUDE}
      -c ${CMAKE_CURRENT_SOURCE_DIR}/${src})
ENDFOREACH(src)


# Link libtcc1
ADD_CUSTOM_COMMAND(OUTPUT libtcc1.a DEPENDS tiny_libmaker ${LIBTCC1_OBJS}
  COMMAND ${CMAKE_CURRENT_BINARY_DIR}/tiny_libmaker rcs libtcc1.a
          ${LIBTCC1_OBJS}
)
ADD_CUSTOM_TARGET(libtcc1 ALL DEPENDS libtcc1.a)


# Install files.
INSTALL(TARGETS tcc RUNTIME DESTINATION ${INSTALL_BIN})
INSTALL(TARGETS libtcc ARCHIVE DESTINATION ${INSTALL_LIB})
INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/libtcc1.a DESTINATION ${INSTALL_LIB}/tcc)
INSTALL(FILES libtcc.h DESTINATION ${INSTALL_INC})
INSTALL(FILES tcc-doc.html DESTINATION ${INSTALL_DOC})
INSTALL(FILES include/stdarg.h include/stddef.h include/stdbool.h include/float.h include/varargs.h include/tcclib.h DESTINATION ${INSTALL_LIB}/tcc/include)
INSTALL(FILES README COPYING DESTINATION ${INSTALL_DATA})
IF(WIN32 AND NOT CYGWIN)
  INSTALL(TARGETS tcc RUNTIME DESTINATION ${INSTALL_BIN})
  INSTALL(FILES
    win32/include/assert.h
    win32/include/conio.h
    win32/include/ctype.h
    win32/include/dir.h
    win32/include/direct.h
    win32/include/dirent.h
    win32/include/dos.h
    win32/include/errno.h
    win32/include/excpt.h
    win32/include/fcntl.h
    win32/include/fenv.h
    win32/include/float.h
    win32/include/inttypes.h
    win32/include/io.h
    win32/include/limits.h
    win32/include/locale.h
    win32/include/malloc.h
    win32/include/math.h
    win32/include/mem.h
    win32/include/memory.h
    win32/include/process.h
    win32/include/setjmp.h
    win32/include/share.h
    win32/include/signal.h
    win32/include/stdarg.h
    win32/include/stdbool.h
    win32/include/stddef.h
    win32/include/stdint.h
    win32/include/stdio.h
    win32/include/stdlib.h
    win32/include/string.h
    win32/include/tchar.h
    win32/include/time.h
    win32/include/unistd.h
    win32/include/values.h
    win32/include/varargs.h
    win32/include/wchar.h
    win32/include/wctype.h
    win32/include/_mingw.h
    DESTINATION ${INSTALL_LIB}/tcc/include)
  INSTALL(FILES
    win32/include/sys/fcntl.h
    win32/include/sys/file.h
    win32/include/sys/locking.h
    win32/include/sys/stat.h
    win32/include/sys/time.h
    win32/include/sys/timeb.h
    win32/include/sys/types.h
    win32/include/sys/unistd.h
    win32/include/sys/utime.h
    DESTINATION ${INSTALL_LIB}/tcc/include/sys)
  INSTALL(FILES
    win32/include/winapi/basetsd.h
    win32/include/winapi/basetyps.h
    win32/include/winapi/winbase.h
    win32/include/winapi/wincon.h
    win32/include/winapi/windef.h
    win32/include/winapi/windows.h
    win32/include/winapi/winerror.h
    win32/include/winapi/wingdi.h
    win32/include/winapi/winnetwk.h
    win32/include/winapi/winnls.h
    win32/include/winapi/winnt.h
    win32/include/winapi/winreg.h
    win32/include/winapi/winsvc.h
    win32/include/winapi/winuser.h
    win32/include/winapi/winver.h
    DESTINATION ${INSTALL_LIB}/tcc/include/winapi)
  INSTALL(FILES
    win32/lib/gdi32.def
    win32/lib/msvcrt.def
    win32/lib/kernel32.def
    win32/lib/user32.def
    DESTINATION ${INSTALL_LIB})
ENDIF(WIN32 AND NOT CYGWIN)
