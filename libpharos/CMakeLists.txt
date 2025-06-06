# pharos library sources
set(PHAROS_SOURCES
  partitioner.cpp               # Placed at the top, as it is a slow compile

  apidb.cpp
  apigraph.cpp
  apisig.cpp
  badcode.cpp
  bua.cpp
  calls.cpp
  cdg.cpp
  config.cpp
  convention.cpp
  defuse.cpp
  demangle.cpp
  demangle_json.cpp
  descriptors.cpp
  funcs.cpp
  globals.cpp
  graph.cpp
  imports.cpp
  ir.cpp
  json.cpp
  limit.cpp
  masm.cpp
  md5.cpp
  memory.cpp
  method.cpp
  misc.cpp
  ooanalyzer.cpp
  ooclass.cpp
  ooelement.cpp
  oomember.cpp
  oomethod.cpp
  oosolver.cpp
  oovftable.cpp
  options.cpp
  path.cpp
  pdg.cpp
  prolog.cpp
  prologimpl.cpp
  prolog_symexp.cpp
  revision.cpp
  riscops.cpp
  semantics.cpp
  spacer.cpp
  sptrack.cpp
  state.cpp
  stkvar.cpp
  swiimpl.cpp
  tags.cpp
  threads.cpp
  typedb.cpp
  types.cpp
  usage.cpp
  util.cpp
  vcall.cpp
  vftable.cpp
  wp.cpp
  yaml.cpp
  znode.cpp

  PHAROS_REVISION.ii
  config.yaml.ii
  tags.yaml.ii
  )

# Packages needed by libpharos
set(swipl_required_version "8.3.16")
find_package(SWIPL "9.0.0" QUIET)
if(NOT SWIPL_FOUND)
  find_package(SWIPL ${swipl_required_version} REQUIRED)
else()
  message(STATUS "Fount SWIPL: ${SWIPL_EXECUTABLE} (found version ${SWIPL_VERSION})")
endif()
find_package(SQLite REQUIRED)
set(CURSES_NEED_NCURSES true)
find_package(Curses REQUIRED)
find_package(Threads)

set(Boost_USE_MULTITHREADED on)
set(_old_FPPC "${CMAKE_FIND_PACKAGE_PREFER_CONFIG}")
set(CMAKE_FIND_PACKAGE_PREFER_CONFIG true)
find_package(Boost 1.60.0 REQUIRED
  COMPONENTS system chrono timer iostreams serialization)
set(CMAKE_FIND_PACKAGE_PREFER_CONFIG "${_old_FPPC}")

# Create pharos library
add_library(pharos SHARED ${PHAROS_SOURCES})
target_link_libraries(pharos PUBLIC
  Threads::Threads Boost::chrono Boost::timer Boost::iostreams Boost::serialization ${CMAKE_DL_LIBS}
  Rose::Rose SQLite::SQLite ${CURSES_LIBRARIES})
target_link_libraries(pharos PRIVATE SQLite::SQLite)
target_link_libraries(pharos PUBLIC swipl::libswipl)
if(COMMAND target_precompile_headers)
  target_precompile_headers(pharos PRIVATE "<rose.h>")
endif()

# The following has been necessary to get things to work on OSX.  This
# is almost certainly wrong on a Windows build, but we don't support
# that yet.
if(CMAKE_USE_PTHREADS_INIT)
  target_compile_options(pharos PUBLIC "-pthread")
endif()

# curses and Rose include dirs
target_include_directories(pharos SYSTEM PUBLIC
  ${CURSES_INCLUDE_DIRS}
  $<TARGET_PROPERTY:Rose::Rose,INTERFACE_INCLUDE_DIRECTORIES>)
# Set this directory as an include for targets that use the pharos library
target_include_directories(pharos INTERFACE
  $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}>
  $<INSTALL_INTERFACE:include/libpharos>)
# Generated include files
target_include_directories(pharos PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<INSTALL_INTERFACE:include/libpharos>)

# install target
install(TARGETS pharos EXPORT libpharos DESTINATION lib)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} DESTINATION include
  FILES_MATCHING PATTERN "*.hpp")
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR} DESTINATION include
  FILES_MATCHING
  PATTERN "*.hpp"
  PATTERN "build.hpp" EXCLUDE)
install(EXPORT libpharos DESTINATION lib/cmake/libpharos
  EXPORT_LINK_INTERFACE_LIBRARIES)

# Create config.yaml
get_filename_component(SWI_INSTALL_DIR "${SWIPL_EXECUTABLE}" DIRECTORY)
configure_file(config.yaml.in config.yaml)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/config.yaml"
  DESTINATION etc
  RENAME pharos.yaml)

# Create version.hpp
rose_version_from_string(${ROSE_VERSION} ROSE_NUMERIC_VERSION)
rose_version_from_string(${PHAROS_ROSE_MINIMUM_VERSION} PHAROS_ROSE_MINIMUM_NUMERIC_VERSION)
configure_file(version.hpp.in version.hpp)

# Generate PHAROS_REVISION in every build
add_custom_target(generate_pharos_revision ALL)
add_custom_command(
  POST_BUILD
  TARGET generate_pharos_revision
  COMMAND ${CMAKE_COMMAND}
  "-Dsrcdir=${CMAKE_CURRENT_SOURCE_DIR}"
  -P "${CMAKE_CURRENT_SOURCE_DIR}/make_pharos_revision.cmake"
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  BYPRODUCTS PHAROS_REVISION)
add_dependencies(pharos generate_pharos_revision)
set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES PHAROS_REVISION)

# Copy PHAROS_REVISION to PHAROS_REVISION.ii if changed
add_custom_command(
  OUTPUT PHAROS_REVISION.ii
  COMMAND ${CMAKE_COMMAND} -E copy_if_different PHAROS_REVISION PHAROS_REVISION.ii
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  MAIN_DEPENDENCY PHAROS_REVISION)
set_source_files_properties("${CMAKE_CURRENT_BINARY_DIR}/PHAROS_REVISION.ii"
  PROPERTIES HEADER_FILE_ONLY true)

# Generate config.yaml.ii from config.yaml
add_custom_command(
  OUTPUT config.yaml.ii
  COMMAND xxd -i config.yaml > "${CMAKE_CURRENT_BINARY_DIR}/config.yaml.ii"
  WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
  MAIN_DEPENDENCY "${CMAKE_CURRENT_BINARY_DIR}/config.yaml")
set_source_files_properties("${CMAKE_CURRENT_BINARY_DIR}/config.yaml.ii"
  PROPERTIES HEADER_FILE_ONLY true)

macro(convert_tags_file tags)
add_custom_command(
  OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${tags}"
  COMMAND sed -e [=[s/#.*$//g; :a; N; s/#.*$//g; $!ba; s/[[:space:]]//g; s/:/: /g]=]
  < "${CMAKE_CURRENT_SOURCE_DIR}/${tags}"
  > "${CMAKE_CURRENT_BINARY_DIR}/${tags}"
  VERBATIM
  WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
  MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/${tags}")
add_custom_command(
  OUTPUT ${tags}.ii
  COMMAND xxd -i ${tags} > ${tags}.ii
  WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
  MAIN_DEPENDENCY "${CMAKE_CURRENT_BINARY_DIR}/${tags}")
endmacro()

convert_tags_file(tags.yaml)
