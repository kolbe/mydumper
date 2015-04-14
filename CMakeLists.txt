cmake_minimum_required(VERSION 2.6)
project(mydumper)
set(VERSION_MAJOR 0)
set(VERSION_MINOR 6)
set(VERSION_PATCH 3)
set(VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
set(ARCHIVE_NAME "${CMAKE_PROJECT_NAME}-${VERSION}")

set(PACKAGE FALSE CACHE BOOL "Enable package building (this disables local installation of system files)")

#Required packages
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/modules)
find_package(MySQL)
find_package(ZLIB)
find_package(GLIB2)
find_package(PCRE)

option(BUILD_DOCS "Build the documentation" ON)

if (BUILD_DOCS)
  add_subdirectory(docs)
endif (BUILD_DOCS)

option(WITH_BINLOG "Build binlog dump options" OFF)

set(CMAKE_C_FLAGS "-Wall -Wno-deprecated-declarations -Wunused -Wwrite-strings -Wno-strict-aliasing -Wextra -Wshadow -Werror -O3 -g ${MYSQL_CFLAGS}")

include_directories(${MYDUMPER_SOURCE_DIR} ${MYSQL_INCLUDE_DIR} ${GLIB2_INCLUDE_DIR} ${PCRE_INCLUDE_DIR} ${ZLIB_INCLUDE_DIRS})

if (NOT CMAKE_INSTALL_PREFIX)
  SET(CMAKE_INSTALL_PREFIX "/usr/local" CACHE STRING "Install path" FORCE)
endif (NOT CMAKE_INSTALL_PREFIX)
MARK_AS_ADVANCED(CMAKE)

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/config.h)

if(PACKAGE)
  #install(FILES ${CMAKE_BINARY_DIR}/mydumper DESTINATION .  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  #install(FILES ${CMAKE_BINARY_DIR}/myloader DESTINATION .  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
  if(${CMAKE_VERSION} VERSION_LESS 2.8.11)
    message(WARNING "CMake version is ${CMAKE_VERSION}. Building of packages requires version 2.8.11 or greater.")
  else()
    # See if we are on a RPM-capable or DEB-capable system
    find_program(RPMBUILD rpmbuild)
    find_program(DEBBUILD dpkg-buildpackage)
    set(CPACK_GENERATOR "TGZ")
    if(NOT ( ${RPMBUILD} STREQUAL "RPMBUILD-NOTFOUND" ) )
      message(STATUS "Generating RPM packages")
      set(CPACK_GENERATOR "${CPACK_GENERATOR};RPM")
    endif()

    if(NOT ( ${DEBBUILD} STREQUAL "DEBBUILD-NOTFOUND" ) )
      set(CPACK_GENERATOR "${CPACK_GENERATOR};DEB")
      execute_process(COMMAND dpgk --print-architecture OUTPUT_VARIABLE
DEB_ARCHITECTURE)
      set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${DEB_ARCHITECTURE})
      set (CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
      message(STATUS "Generating DEB packages for ${DEB_ARCHITECTURE}")
    endif()
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "mydumper")
    set(CPACK_PACKAGE_VERSION_MAJOR "${VERSION_MAJOR}")
    set(CPACK_PACKAGE_VERSION_MINOR "${VERSION_MINOR}")
    set(CPACK_PACKAGE_VERSION_PATCH "${VERSION_PATCH}")
    set(CPACK_PACKAGE_CONTACT "MariaDB Corporation Ab")
    set(CPACK_PACKAGE_NAME "mydumper")
    if (OSVERSION)
        set(OSVERSION "${OSVERSION}-")
    else()
        set(OSVERSION "")
    endif()
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${VERSION}-${OSVERSION}${CMAKE_SYSTEM_PROCESSOR}")
    set(CPACK_PACKAGE_VENDOR "MariaDB Corporation Ab")
    #set(CPACK_PACKAGE_DESCRIPTION_FILE ${CMAKE_SOURCE_DIR}/etc/DESCRIPTION)
    set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
    #set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_BINARY_DIR}/postinst;{CMAKE_BINARY_DIR}/postrm")
    #set(CPACK_RPM_POST_INSTALL_SCRIPT_FILE ${CMAKE_BINARY_DIR}/postinst)
    #set(CPACK_RPM_POST_UNINSTALL_SCRIPT_FILE ${CMAKE_BINARY_DIR}/postrm)
    set(CPACK_RPM_PACKAGE_NAME "mydumper")
    set(CPACK_RPM_PACKAGE_VENDOR "MariaDB Corporation Ab")
    set(CPACK_RPM_PACKAGE_LICENSE "GPLv2")
    set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/etc /etc/ld.so.conf.d /etc/init.d /etc/rc.d/init.d")
    set(CPACK_RPM_SPEC_MORE_DEFINE "%define ignore \#")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc/init.d")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc/ld.so.conf.d")
    set(CPACK_RPM_USER_FILELIST "%ignore /etc")
    include(CPack)
  endif()
endif()

if (WITH_BINLOG)
  add_executable(mydumper mydumper.c binlog.c server_detect.c g_unix_signal.c)
else (WITH_BINLOG)
  add_executable(mydumper mydumper.c server_detect.c g_unix_signal.c)
endif (WITH_BINLOG)
target_link_libraries(mydumper ${MYSQL_LIBRARIES} ${GLIB2_LIBRARIES} ${GTHREAD2_LIBRARIES} ${PCRE_PCRE_LIBRARY} ${ZLIB_LIBRARIES})


add_executable(myloader myloader.c)
target_link_libraries(myloader ${MYSQL_LIBRARIES} ${GLIB2_LIBRARIES} ${GTHREAD2_LIBRARIES} ${PCRE_PCRE_LIBRARY} ${ZLIB_LIBRARIES})

INSTALL(TARGETS mydumper myloader
  RUNTIME DESTINATION bin
)

add_custom_target(dist
  COMMAND bzr export --root=${ARCHIVE_NAME}
    ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.gz
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

OPTION(RUN_CPPCHECK "Run cppcheck" OFF)

IF(RUN_CPPCHECK)
  include(CppcheckTargets)
  add_cppcheck(mydumper)
  add_cppcheck(myloader)
ENDIF(RUN_CPPCHECK)


MESSAGE(STATUS "------------------------------------------------")
MESSAGE(STATUS "MYSQL_CONFIG = ${MYSQL_CONFIG}")
MESSAGE(STATUS "CMAKE_INSTALL_PREFIX = ${CMAKE_INSTALL_PREFIX}")
MESSAGE(STATUS "BUILD_DOCS = ${BUILD_DOCS}")
MESSAGE(STATUS "WITH_BINLOG = ${WITH_BINLOG}")
MESSAGE(STATUS "RUN_CPPCHECK = ${RUN_CPPCHECK}")
MESSAGE(STATUS "Change a values with: cmake -D<Variable>=<Value>")
MESSAGE(STATUS "------------------------------------------------")
MESSAGE(STATUS)

