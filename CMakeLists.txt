#cmake_minimum_required(VERSION 2.8 FATAL_ERROR)
cmake_minimum_required(VERSION 3.14)
PROJECT(sioclient)

option(BUILD_SHARED_LIBS "Build the shared library" OFF)
option(BUILD_UNIT_TESTS  "Builds unit tests target" OFF)

set(MAJOR 1)
set(MINOR 6)
set(PATCH 0)

if(NOT BORINGSSL_INCLUDE_PATH)
	set(BORINGSSL_INCLUDE_PATH "" CACHE STRING "boringssl include path")
endif()
message("\n=========== sioclient Build Configuration ===========\n")
message(STATUS "BUILD_SHARED_LIBS          : " ${BUILD_SHARED_LIBS})
message(STATUS "BUILD_UNIT_TESTS           : " ${BUILD_UNIT_TESTS})
message(STATUS "BORINGSSL_INCLUDE_PATH     : " ${BORINGSSL_INCLUDE_PATH})


#add_compile_options(
#		$<$<CONFIG:Debug>:-D_GLIBCXX_DEBUG>
#)
if (MSVC)
	set(CMAKE_CXX_FLAGS_DEBUG "/Zi /Ob0 /Od /RTC1 /MTd")
	set(CMAKE_CXX_FLAGS_RELEASE  "/O2 /Ob2 /DNDEBUG /MT")

	message(STATUS "CMAKE_CXX_FLAGS_DEBUG : " ${CMAKE_CXX_FLAGS_DEBUG})
	message(STATUS "CMAKE_CXX_FLAGS_RELEASE : " ${CMAKE_CXX_FLAGS_RELEASE})

	add_compile_options(
			/bigobj
			$<$<CONFIG:>:/MT> #---------|
			$<$<CONFIG:Debug>:/MTd> #---|-- Statically link the runtime libraries
			$<$<CONFIG:Release>:/MT> #--|
	)
else ()
	add_definitions(
			-std=c++14
	)
endif()

if(NOT CMAKE_BUILD_TYPE )
MESSAGE(STATUS "not define build type, set to release" )
set(CMAKE_BUILD_TYPE Release )
elseif(NOT (${CMAKE_BUILD_TYPE} STREQUAL "Release" OR ${CMAKE_BUILD_TYPE} STREQUAL "Debug" ))
MESSAGE(SEND_ERROR "CMAKE_BUILD_TYPE must be either Release or Debug")
return()
endif()

aux_source_directory(${CMAKE_CURRENT_LIST_DIR}/src ALL_SRC)
aux_source_directory(${CMAKE_CURRENT_LIST_DIR}/src/internal ALL_SRC)
file(GLOB ALL_HEADERS ${CMAKE_CURRENT_LIST_DIR}/src/*.h )
set(SIO_INCLUDEDIR ${CMAKE_CURRENT_LIST_DIR})

add_definitions(
    # These will force ASIO to compile without Boost
    -DBOOST_DATE_TIME_NO_LIB
    -DBOOST_REGEX_NO_LIB
    -DASIO_STANDALONE
    # These will force WebsocketPP to compile with C++11
    -D_WEBSOCKETPP_CPP11_STL_
    -D_WEBSOCKETPP_CPP11_FUNCTIONAL_
)

add_library(sioclient ${ALL_SRC})
target_include_directories(sioclient PRIVATE 
    ${CMAKE_CURRENT_LIST_DIR}/src 
    ${CMAKE_CURRENT_LIST_DIR}/lib/websocketpp 
    ${CMAKE_CURRENT_LIST_DIR}/lib/rapidjson/include
    ${CMAKE_CURRENT_LIST_DIR}/lib/asio/asio/include
)

if (CMAKE_VERSION VERSION_GREATER "3.1")
set_property(TARGET sioclient PROPERTY CXX_STANDARD 11)
set_property(TARGET sioclient PROPERTY CXX_STANDARD_REQUIRED ON)
else()
set_property(TARGET sioclient APPEND_STRING PROPERTY COMPILE_FLAGS "-std=c++11")
endif()
if(BUILD_SHARED_LIBS)
set_target_properties(sioclient
	PROPERTIES
		SOVERSION ${MAJOR}
		VERSION ${MAJOR}.${MINOR}.${PATCH}
	)
endif()
list(APPEND TARGET_LIBRARIES sioclient)

#find_package(OpenSSL)
#if(OPENSSL_FOUND)
add_library(sioclient_tls ${ALL_SRC})
target_include_directories(sioclient_tls PRIVATE
    ${CMAKE_CURRENT_LIST_DIR}/src 
    ${CMAKE_CURRENT_LIST_DIR}/lib/websocketpp 
    ${CMAKE_CURRENT_LIST_DIR}/lib/rapidjson/include
    ${CMAKE_CURRENT_LIST_DIR}/lib/asio/asio/include
#    ${OPENSSL_INCLUDE_DIR}
    ${BORINGSSL_INCLUDE_PATH}
)

if (CMAKE_VERSION VERSION_GREATER "3.1")
set_property(TARGET sioclient_tls PROPERTY CXX_STANDARD 14)
set_property(TARGET sioclient_tls PROPERTY CXX_STANDARD_REQUIRED ON)
target_link_libraries(sioclient_tls PRIVATE ${OPENSSL_LIBRARIES} )
else()
set_property(TARGET sioclient_tls APPEND_STRING PROPERTY COMPILE_FLAGS "-std=c++14")
endif()
#target_compile_definitions(sioclient_tls PRIVATE -DSIO_TLS)
target_compile_definitions(sioclient_tls PRIVATE
		SIO_TLS
		$<$<PLATFORM_ID:Windows>:WIN32_LEAN_AND_MEAN>
)
if(BUILD_SHARED_LIBS)
set_target_properties(sioclient_tls
	PROPERTIES
		SOVERSION ${MAJOR}
		VERSION ${MAJOR}.${MINOR}.${PATCH}
	)
endif()
list(APPEND TARGET_LIBRARIES sioclient_tls)

#endif()

include(GNUInstallDirs)

install(FILES ${ALL_HEADERS} 
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(TARGETS ${TARGET_LIBRARIES}
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(BUILD_UNIT_TESTS)
message(STATUS "Building with unit test support.")
enable_testing()
add_subdirectory(test)
endif()
