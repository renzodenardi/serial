cmake_minimum_required(VERSION 2.8.3)
project(serial)


if(APPLE)
	find_library(IOKIT_LIBRARY IOKit)
	find_library(FOUNDATION_LIBRARY Foundation)
endif()

## Overide with cmake -DCMAKE_BUILD_TYPE=Debug {dir}
if( NOT CMAKE_BUILD_TYPE AND NOT _WIN_ )
   message("Build type not set (defaults to release)")
   message("-DCMAKE_BUILD_TYPE=Debug for debug")
   set( CMAKE_BUILD_TYPE Release )
endif()

string(TOLOWER ${PROJECT_NAME} LIBRARY_NAME)

## Make an uninstall target 
configure_file(
 	"${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  	"${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  	IMMEDIATE @ONLY
)

add_custom_target(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")

## Sources
set(serial_SRCS
    src/serial.cc
    include/serial/serial.h
    include/serial/v8stdint.h
)
if(APPLE)
    # If OSX
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_osx.cc)
elseif(UNIX)
    # If unix
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_linux.cc)
else()
    # If windows
    list(APPEND serial_SRCS src/impl/win.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_win.cc)
endif()

## Add serial library
add_library(${PROJECT_NAME} SHARED ${serial_SRCS})
if(APPLE)
	target_link_libraries(${PROJECT_NAME} ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
elseif(UNIX)
	target_link_libraries(${PROJECT_NAME} rt pthread)
else()
	target_link_libraries(${PROJECT_NAME} setupapi)
endif()

## Include headers
include_directories(include)

## Examples
if(BUILD_EXAMPLES)
  add_executable(serial_example examples/serial_example.cc)
  add_dependencies(serial_example ${PROJECT_NAME})
  target_link_libraries(serial_example ${PROJECT_NAME})
endif()

## Tests
if(BUILD_TESTS)
    add_subdirectory(tests)
endif()

################################################################################

# Create variables used for exporting (from build tree) in serialConfig.cmake
SET( serial_LIBRARIES ${LIBRARY_NAME} )
SET( serial_INCLUDE_DIR ${PROJECT_SOURCE_DIR} )

# Export package for use from the build tree
EXPORT( PACKAGE ${PROJECT_NAME} )

# Create the serialConfig.cmake file for other cmake projects.
# ... for the build tree
SET( CONFIG_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
SET( CONFIG_DIR ${CMAKE_CURRENT_BINARY_DIR})
CONFIGURE_FILE( ${CMAKE_CURRENT_SOURCE_DIR}/serialConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/serialConfig.cmake @ONLY )
# ... for the install tree
SET( CMAKECONFIG_INSTALL_DIR lib/cmake/${PROJECT_NAME} )
FILE( RELATIVE_PATH REL_INCLUDE_DIR
    "${CMAKE_INSTALL_PREFIX}/${CMAKECONFIG_INSTALL_DIR}"
    "${CMAKE_INSTALL_PREFIX}/include" )

SET( serial_INCLUDE_DIR "\${serial_CMAKE_DIR}/${REL_INCLUDE_DIR}" )
SET( CONFIG_SOURCE_DIR )
SET( CONFIG_DIR )
CONFIGURE_FILE( ${CMAKE_CURRENT_SOURCE_DIR}/serialConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/serialConfig.cmake @ONLY )

INSTALL(FILES "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/serialConfig.cmake"
        DESTINATION ${CMAKECONFIG_INSTALL_DIR} )
install(TARGETS ${LIBRARY_NAME}
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
)
# Install headers
INSTALL(FILES include/serial/serial.h include/serial/v8stdint.h
  DESTINATION ${CMAKE_INSTALL_PREFIX}/include/${PROJECT_NAME})
