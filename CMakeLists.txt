if (CMAKE_USE_OPENSSL)
    find_package(OpenSSL REQUIRED)
    set(USE_OPENSSL "-DUSE_OPENSSL -DLIBUS_USE_OPENSSL")
    #set(OPENSSL_LIB "OpenSSL::SSL OpenSSL::Crypto")
    list(APPEND CMAKE_REQUIRED_INCLUDES ${OPENSSL_INCLUDE_DIR})
else()
    set(NOT_USE_OPENSSL "-DLIBUS_NO_SSL")
endif()

#find_package(libuv CONFIG REQUIRED)
#if (TARGET libuv::uv)
#    set(LIBUV_LIBRARY libuv::uv)
#else()
#    set(LIBUV_LIBRARY libuv::uv_a)
#endif()

file(GLOB SOURCES src/*.c src/eventing/*.c)

set(USOCKETS_EXT_INCLUDE_DIR )
set(USOCKETS_EXT_LIBS )

if (CMAKE_USE_OPENSSL)
    # It requires C++17 or later, see https://github.com/uNetworking/uSockets/blob/0ebdde0601cc82349fc11a7c4bbb6dc5c9f28f42/Makefile#L55
    set(CMAKE_CXX_STANDARD 17)
    find_package(OpenSSL REQUIRED)
    file(GLOB SSL_SOURCES src/crypto/*.c*)
    list(APPEND SOURCES ${SSL_SOURCES})
    list(APPEND USOCKETS_EXT_LIBS OpenSSL::SSL OpenSSL::Crypto)
endif()

if (CMAKE_USE_EVENT)
    file(GLOB SSL_SOURCES src/eventing/*.c)
    list(APPEND SOURCES ${SSL_SOURCES})
    list(APPEND USOCKETS_EXT_INCLUDE_DIR src/internal/eventing)
endif()

if (CMAKE_USE_NETWORK)
    list(APPEND USOCKETS_EXT_INCLUDE_DIR src/internal/networking)
    list(APPEND USOCKETS_EXT_LIBS ws2_32)
endif()

add_library(uSockets ${SOURCES})

if (${LIBUS_USE_LIBUV})
    target_compile_definitions(uSockets PRIVATE -DLIBUS_USE_LIBUV)
endif()

target_compile_definitions(uSockets PRIVATE ${NOT_USE_OPENSSL} ${USE_OPENSSL})
#target_include_directories(uSockets PUBLIC ${OPENSSL_INCLUDE_DIR} ${USOCKETS_EXT_INCLUDE_DIR} PRIVATE "${CMAKE_CURRENT_LIST_DIR}/src")
target_link_libraries(uSockets PUBLIC ${OPENSSL_LIBRARIES} ${LIBUV_LIBRARY} ${USOCKETS_EXT_LIBS})

#MESSAGE(WARNING "USOCKETS_EXT_INCLUDE_DIR: " ${USOCKETS_EXT_INCLUDE_DIR})

set(TARGETS_EXPORT_NAME "uSocketsTargets")
set(CONFIG_EXPORT_NAME "uSocketsConfig")

source_group( # IDEs should put the headers in a nice place
        TREE "${CMAKE_CURRENT_SOURCE_DIR}/src"
        PREFIX "Header Files"
        FILES ${HEADERS})

option(BUILD_SHARED_LIBS "Build shared library" ON)

target_include_directories(uSockets
        PUBLIC
        $<BUILD_INTERFACE:${OPENSSL_INCLUDE_DIR}>
#        $<BUILD_INTERFACE:${USOCKETS_EXT_INCLUDE_DIR}>

        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src> ## makes it #include "libsockets.h"
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/crypto>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/eventing>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/internal/eventing>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/internal/networking>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/io_uring>

#        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src>
#        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
#        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/src>

#        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_INCLUDEDIR}>
        $<INSTALL_INTERFACE:${OPENSSL_INCLUDE_DIR}>
        $<INSTALL_INTERFACE:${USOCKETS_EXT_INCLUDE_DIR}>
#        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/src" #/src
)

install(DIRECTORY
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

include(GNUInstallDirs) # make cache variables for install destinations

install(TARGETS uSockets
        EXPORT ${TARGETS_EXPORT_NAME}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/uSockets")

######
include(CMakePackageConfigHelpers)

install(EXPORT ${TARGETS_EXPORT_NAME}
        FILE "${TARGETS_EXPORT_NAME}.cmake"
        NAMESPACE uSockets::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uSockets
)

export(EXPORT ${TARGETS_EXPORT_NAME}
        FILE "${CMAKE_BINARY_DIR}/cmake/${TARGETS_EXPORT_NAME}.cmake"
        NAMESPACE uSockets::
)

install(FILES
        "${CMAKE_BINARY_DIR}/cmake/${CONFIG_EXPORT_NAME}.cmake"
        "${CMAKE_BINARY_DIR}/cmake/${CONFIG_EXPORT_NAME}-version.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uSockets
)

