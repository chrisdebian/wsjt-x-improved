cmake_minimum_required (VERSION 2.8.10 FATAL_ERROR)

project (wsjt-superbuild)

set (__default_tag master)
set (__hamlib_upstream hamlib-4.7)

set (TARGET_DIR wsjtx
  CACHE STRING "Tarball name and top level directory therein

If not specified the WSJT_TAG variable will be substituted" )

set (WSJTX_TAG ${__default_tag}
  CACHE STRING "WSJT-X tag to build, if unset build the latest master branch HEAD.

For a tag use <tag-name> e.g. wsjtx-3.0.0

The  WSJTX_TAG option  is  only used  if the  directory  'src' is  not
present.   If the  'src' directory  is present  the build  expects the
sources to be found in it consisting of the two files 'hamlib.tgz' and
'wsjtx.tgz'.  This 'src' directory is to  be used where build hosts do
not allow  source downloads using Subversion,  git or cURL as  part of
the build.  See the  'source' target  for how  to generate  a suitable
source tarball." )

set (HAMLIB_TAG ${__default_tag}
  CACHE STRING "Hamlib tag to build, if unset and WSJTX_TAG is not defined build the latest master branch HEAD.

For a tag use <tag-name> e.g. wsjtx-3.0.0 or any other git reference

The HAMLIB_TAG will default to the WSJTX_TAG if not specified.

The HAMLIB_TAG option is only used if the directory 'src' is not
present." )


#
# find prerequisites
#
# Find_library (USB_LIBRARY NAMES libusb.a usb)
Find_program (PATCH_EXECUTABLE patch REQUIRED)
Find_package (Git REQUIRED)

#
# extra C flags to minimize hamlib excutable sizes
#
if (NOT APPLE)
  set (EXTRA_FLAGS "CFLAGS=-g -O2 -fPIC -fdata-sections -ffunction-sections" "LDFLAGS=-Wl,--gc-sections")
endif (NOT APPLE)


#
# function to read and extract MD5 hashes from a file
#
function (read_md5sum file result_variable)
  file (READ ${file}.md5sum md5sum)
  string (REGEX REPLACE "^([^ ]+).*$" "\\1" md5sum ${md5sum})
  set (${result_variable} ${md5sum} PARENT_SCOPE)
endfunction (read_md5sum)


#
# work out what we want to build
#
if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src")
  # one day this needs to change to the hamlib git master
  set (hamlib_repo git://git.code.sf.net/u/bsomervi/hamlib)

  set (wsjtx_repo git@bitbucket.org:k1jt/wsjtx.git)

  # one day this needs to change to master or some agreed stable SHA
  set (hamlib_TAG integration)

  # decide if we want to build from master or a tag
  if (NOT ${WSJTX_TAG} STREQUAL ${__default_tag})
    # assume we have a tag so find matching Hamlib tag unless
    # HAMLIB_TAG has been specified
    #
    # otehrwise it is assumed that a tag exists in the Hamlib
    # repository that is exactly the same as the WSJT-X repository tag
    # we intend to build from
    if (NOT ${HAMLIB_TAG} STREQUAL ${__default_tag})
      set (hamlib_TAG "${HAMLIB_TAG}")
    else ()
      set (hamlib_TAG "${WSJTX_TAG}")
    endif ()
    if (NOT ${TARGET_DIR} STREQUAL wsjtx)
      set (target_dir "${TARGET_DIR}")
    else ()
      set (target_dir "${WSJTX_TAG}")
    endif ()
    message (STATUS "Building WSJT-X tag: ${WSJTX_TAG}")
    message (STATUS "hamlib tag: ${hamlib_TAG}")
  else ()
    set (target_dir wsjtx)
    message (STATUS "Building WSJT-X branch: ${WSJTX_TAG}")
    message (STATUS "hamlib branch: ${hamlib_TAG}")
  endif ()
else (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src")
  read_md5sum (src/${__hamlib_upstream}.tar.gz hamlib_md5sum)
  read_md5sum (src/wsjtx.tgz wsjtx_md5sum)
endif (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src")


include (ExternalProject)


#
# build and install hamlib locally so it can be referenced by the
# WSJT-X build
#
ExternalProject_Add (hamlib
  GIT_REPOSITORY ${hamlib_repo}
  GIT_TAG ${hamlib_TAG}
  GIT_SHALLOW False
  URL ${CMAKE_CURRENT_SOURCE_DIR}/src/${__hamlib_upstream}.tar.gz
  URL_HASH MD5=${hamlib_md5sum}
  #UPDATE_COMMAND ${CMAKE_COMMAND} -E env "[ -f ./bootstrap ] && ./bootstrap"
  PATCH_COMMAND ${PATCH_EXECUTABLE} -p1 -N < ${CMAKE_CURRENT_SOURCE_DIR}/hamlib.patch
  CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=<INSTALL_DIR> --disable-shared --enable-static --without-cxx-binding ${EXTRA_FLAGS} # LIBUSB_LIBS=${USB_LIBRARY}
  BUILD_COMMAND $(MAKE) all V=1 # $(MAKE) is ExternalProject_Add() magic to do recursive make
  INSTALL_COMMAND $(MAKE) install-strip V=1 DESTDIR=""
  STEP_TARGETS update install
  )

#
# custom target to make a hamlib source tarball
#
add_custom_target (hamlib_sources
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/build
  COMMAND ${GIT_EXECUTABLE} clone --branch=${hamlib_TAG} --single-branch ${hamlib_repo} ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/src
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/src ./bootstrap
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/build ../src/configure
  COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/build $(MAKE) dist
  COMMAND ${CMAKE_COMMAND} -E rename ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream/build/${__hamlib_upstream}*.tar.gz ${CMAKE_CURRENT_BINARY_DIR}/${__hamlib_upstream}.tar.gz
  # COMMAND ${GIT_EXECUTABLE} archive --format=tgz --prefix=hamlib/ --remote=${hamlib_repo} ${hamlib_TAG} -o ${CMAKE_CURRENT_BINARY_DIR}/hamlib.tgz
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_CURRENT_BINARY_DIR}/hamlib-upstream
  COMMENT "Generating the hamlib upstream source tarball"
  )


#
# build a list of command line argumenst to pass on to the WSJT-X
# build
#
get_cmake_property (CACHE_VARS CACHE_VARIABLES)
foreach (CACHE_VAR ${CACHE_VARS})
  get_property (CACHE_VAR_HELPSTRING CACHE ${CACHE_VAR} PROPERTY HELPSTRING)
  if (CACHE_VAR_HELPSTRING STREQUAL "No help, variable specified on the command line.")
    get_property (CACHE_VAR_TYPE CACHE ${CACHE_VAR} PROPERTY TYPE)
    if (CACHE_VAR_TYPE STREQUAL "UNINITIALIZED")
      set (CACHE_VAR_TYPE)
    else ()
      set (CACHE_VAR_TYPE :${CACHE_VAR_TYPE})
    endif ()
    list (APPEND CMAKE_ARGS "-D${CACHE_VAR}${CACHE_VAR_TYPE}=${${CACHE_VAR}}")
  endif ()
endforeach ()

#
# build and optionally install WSJT-X using the hamlib package built
# above
#
ExternalProject_Get_Property (hamlib INSTALL_DIR)
ExternalProject_Add (wsjtx
  GIT_REPOSITORY ${wsjtx_repo}
  GIT_TAG ${WSJTX_TAG}
  GIT_SHALLOW False
  URL ${CMAKE_CURRENT_SOURCE_DIR}/src/wsjtx.tgz
  URL_HASH MD5=${wsjtx_md5sum}
  PATCH_COMMAND ${PATCH_EXECUTABLE} -p1 -N < ${CMAKE_CURRENT_SOURCE_DIR}/wsjtx.patch
  CMAKE_ARGS
  ${CMAKE_ARGS}
  -DCMAKE_PREFIX_PATH=${INSTALL_DIR}
  -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
  ${WSJTX_EXTRA_CMAKE_OPTIONS}
  INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
  STEP_TARGETS update configure build install package
  )
ExternalProject_Add_Step (wsjtx package
  COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --target package
  COMMAND echo "Built hamlib from branch/tag: ${hamlib_TAG}"
  COMMAND echo "Built WSJT-X from: ${WSJTX_TAG}"
  COMMAND echo "Package(s) generated in ${BINARY_DIR}."
  COMMENT "Generating WSJT-X package."
  DEPENDEES build
  )

set_target_properties (hamlib PROPERTIES EXCLUDE_FROM_ALL 1)
set_target_properties (wsjtx PROPERTIES EXCLUDE_FROM_ALL 1)

add_dependencies (wsjtx-configure hamlib-install)
add_dependencies (wsjtx-build hamlib-install)
add_dependencies (wsjtx-install hamlib-install)
add_dependencies (wsjtx-package hamlib-install)

# export traditional targets
add_custom_target (build ALL DEPENDS wsjtx-build)
add_custom_target (install DEPENDS wsjtx-install)
add_custom_target (package DEPENDS wsjtx-package)


#
# custom target to make WSJT-X source tarball
#
add_custom_target (wsjtx_sources
  COMMAND ${GIT_EXECUTABLE} archive --format=tgz --prefix=wsjtx/ --remote=${wsjtx_repo} ${WSJTX_TAG} -o ${CMAKE_CURRENT_BINARY_DIR}/wsjtx.tgz
  COMMENT "Generating the WSJT-X upstream source tarball"
  )


#
# custom target to build a self-contained source tarball suitable for
# a local build
#
add_custom_target (source
  COMMAND ${CMAKE_COMMAND} -E make_directory ${target_dir}
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR} ${target_dir}
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${target_dir}/.svn
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${target_dir}/.git
  COMMAND ${CMAKE_COMMAND} -E make_directory ${target_dir}/src
  COMMAND ${CMAKE_COMMAND} -E rename ${__hamlib_upstream}*.tar.gz ${target_dir}/src/${__hamlib_upstream}.tar.gz
  COMMAND ${CMAKE_COMMAND} -E rename wsjtx.tgz ${target_dir}/src/wsjtx.tgz
  COMMAND ${CMAKE_COMMAND} -E md5sum ${target_dir}/src/${__hamlib_upstream}.tar.gz > ${target_dir}/src/${__hamlib_upstream}.tar.gz.md5sum
  COMMAND ${CMAKE_COMMAND} -E md5sum ${target_dir}/src/wsjtx.tgz > ${target_dir}/src/wsjtx.tgz.md5sum
  COMMAND ${CMAKE_COMMAND} -E tar czf ${target_dir}.tgz ${target_dir}
  COMMAND ${CMAKE_COMMAND} -E remove_directory ${target_dir}
  WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
  DEPENDS hamlib_sources wsjtx_sources
  COMMENT "

Generating the WSJT-X superbuild self-contained source tarball ${target_dir}.tgz

To use this tarball, transfer it to the target build host, un-compress
and un-tar it;  then configure it including  a CMAKE_INSTALL_PREFIX if
required  (the default  is /usr/local),  then build  and install.  For
example:

    $ tar xzf ${target_dir}.tgz
    $ mkdir build
    $ cd build
    $ cmake <extra-args> ../${target_dir}
    $ cmake --build .
    $ sudo cmake --build . --target install

where <extra-args>  are CMake  command line arguments  that are  to be
passed  onto  the  WSJT-X  CMake  configuration.  The  sort  of  extra
arguments you might want to pass are for example:

  -D WSJT_MANPAGE_DESTINATION=.

which   changes    the   install   path   for    the   manpages   from
<install-prefix>/share to  <install-prefix>.  This  particular example
might be useful on FreeBSD where manpages are expected to reside under
/usr/local/man.

A test install may be carried out in a local user directory by passing
the DESTDIR variable to the build step e.g.:

    $ cmake --build . --target install -- -j2 DESTDIR=$HOME/local

which would install into ~/local/usr/local.

" )
