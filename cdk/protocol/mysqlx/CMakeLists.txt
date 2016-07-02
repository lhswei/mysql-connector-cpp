# Copyright (c) 2015, 2016, Oracle and/or its affiliates. All rights reserved.
#
# This code is licensed under the terms of the GPLv2
# <http://www.gnu.org/licenses/old-licenses/gpl-2.0.html>, like most
# MySQL Connectors. There are special exceptions to the terms and
# conditions of the GPLv2 as it is applied to this software, see the
# FLOSS License Exception
# <http://www.mysql.com/about/legal/licensing/foss-exception.html>.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published
# by the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
# or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License
# for more details.
#
# You should have received a copy of the GNU General Public License along
# with this program; if not, write to the Free Software Foundation, Inc.,
# 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA


cmake_minimum_required(VERSION 2.8)

include(CheckIncludeFile)

check_include_file(sys/endian.h HAVE_ENDIAN_H)
ADD_CONFIG(HAVE_ENDIAN_H)

check_include_file(sys/byteorder.h HAVE_BYTEORDER_H)  # on Solaris
ADD_CONFIG(HAVE_BYTEORDER_H)

option(DEBUG_PROTOBUF "Debug Protobuf messages (needs WITH_TESTS=1)" OFF)
mark_as_advanced(DEBUG_PROTOBUF)

set(use_full_protobuf ${WITH_TESTS})

if (DEBUG_PROTOBUF)
  message("Protobuf debugging enabled")
  add_definitions(-DDEBUG_PROTOBUF)
  set(use_full_protobuf 1)
endif(DEBUG_PROTOBUF)

if(use_full_protobuf)
  USE_FULL_PROTOBUF()
endif()

SET(PROTOCOL pb)

SET(proto_mysqlx_defs
  ${PROTOCOL}/mysqlx.proto
  ${PROTOCOL}/mysqlx_connection.proto
  ${PROTOCOL}/mysqlx_datatypes.proto
  ${PROTOCOL}/mysqlx_expr.proto
  ${PROTOCOL}/mysqlx_crud.proto
  ${PROTOCOL}/mysqlx_sql.proto
  ${PROTOCOL}/mysqlx_resultset.proto
  ${PROTOCOL}/mysqlx_session.proto
  ${PROTOCOL}/mysqlx_expect.proto
  ${PROTOCOL}/mysqlx_notice.proto
)

if(NOT use_full_protobuf)

#
# Generate modified protobuf sources with LITE_RUNTIME option
# TODO: Find a better way...
#

set(LITE "${CMAKE_CURRENT_BINARY_DIR}/lite")
file(MAKE_DIRECTORY ${LITE})

foreach(proto ${proto_mysqlx_defs})
  get_filename_component(name ${proto} NAME)
  #message("Generating lite version of protobuf source: ${name}")
  file(READ "${proto}" CONTENTS)
  string(REPLACE
    "syntax = \"proto2\""
    "syntax = \"proto2\";\noption optimize_for = LITE_RUNTIME"
    CONTENTS "${CONTENTS}")
  file(WRITE "${LITE}/${name}" "${CONTENTS}")
  #configure_file(lite_template.in "${LITE}/${name}" @ONLY)
  list(APPEND proto_mysqlx_defs_lite "${LITE}/${name}")
endforeach()

set(proto_mysqlx_defs ${proto_mysqlx_defs_lite})

endif()


SET(target_proto_mysqlx ${cdk_target_prefix}proto_mysqlx
    CACHE INTERNAL "CDK proto_mysql target")


ADD_LIBRARY(${target_proto_mysqlx} STATIC
            protocol.cc session.cc rset.cc stmt.cc crud.cc)
ADD_PROTOBUF(${target_proto_mysqlx} ${proto_mysqlx_defs})
TARGET_LINK_LIBRARIES(${target_proto_mysqlx} ${target_foundation})


IF(WIN32)
  TARGET_LINK_LIBRARIES(${target_proto_mysqlx} ws2_32)
ENDIF()

ADD_SUBDIRECTORY(tests)