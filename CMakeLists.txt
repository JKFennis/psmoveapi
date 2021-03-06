
cmake_minimum_required(VERSION 2.6)

project(PSMOVEAPI)

include_directories(${PSMOVEAPI_SOURCE_DIR}/external/hidapi/hidapi)

set(PSMOVEAPI_MAJOR_VERSION "2")
set(PSMOVEAPI_MINOR_VERSION "0")
set(PSMOVEAPI_PATCH_VERSION "0")
set(PSMOVEAPI_VERSION "${PSMOVEAPI_MAJOR_VERSION}.${PSMOVEAPI_MINOR_VERSION}.${PSMOVEAPI_PATCH_VERSION}")
set(PSMOVEAPI_VERSION_SONAME "${PSMOVEAPI_MAJOR_VERSION}")

set(PSMOVEAPI_REQUIRED_LIBS)
set(PSMOVEAPI_PLATFORM_SRC)

set(PSMOVEAPI_TRACKER_REQUIRED_LIBS)
set(PSMOVEAPI_TRACKER_PLATFORM_SRC)

set(PSMOVEAPI_INSTALL_TARGETS)

# Put any other libraries that you might need in here
link_directories(${PSMOVEAPI_SOURCE_DIR}/external/libs)

# Language bindings
option(PSMOVE_BUILD_QT_BINDINGS "Build the Qt bindings" OFF)
option(PSMOVE_BUILD_JAVA_BINDINGS "Build the Java bindings" ON)
option(PSMOVE_BUILD_PROCESSING_BINDINGS "Build the Processing bindings" ON)
option(PSMOVE_BUILD_PYTHON_BINDINGS "Build the Python bindings" ON)


# Build the OpenCV-based camera tracking module? (see psmove_tracker.h)
option(PSMOVE_BUILD_TRACKER "Build the Tracker module (needs OpenCV)" ON)

# Additional applications that can be used to test the library
option(PSMOVE_BUILD_EXAMPLES "Build the C examples" ON)
option(PSMOVE_BUILD_TESTS "Build the C tests" OFF)
option(PSMOVE_BUILD_TUIO_SERVER "Build the TUIO server" OFF)


# Enable tweaks (e.g. registry settings on Windows, ...) for PS Eye
option(PSMOVE_USE_PSEYE "Enable tweaks for the PS Eye camera" ON)

# Build with support for deinterlacing interlaced HD video input
option(PSMOVE_USE_DEINTERLACE "Deinterlace input video (for HD inputs)" OFF)

# Use the CL Eye SDK to interface with the PS Eye camera (Windows only)
option(PSMOVE_USE_CL_EYE_SDK "Use the CL Eye SDK driver on Windows" OFF)

# Make a debug build with helpful output for debugging / maintenance
option(PSMOVE_USE_DEBUG "Build for debugging" OFF)

# Write debugging images for tracking calibration to home directory
option(PSMOVE_USE_TRACKER_TRACE "Write tracker calibration images to disk" OFF)

# Debugging output
if(PSMOVE_USE_DEBUG)
    add_definitions(-DPSMOVE_DEBUG)
    add_definitions(-g -O0 -Wall)
    set(INFO_USE_DEBUG "Yes")
else()
    add_definitions(-O2 -Wall)
    set(INFO_USE_DEBUG "No")
endif()

# Linker commands to put into the .pc file (when installing)
set(PSMOVEAPI_PKGCONFIG_LIBS "-lpsmoveapi")

# Tracker support
if(PSMOVE_BUILD_TRACKER)
    FIND_PACKAGE(OpenCV QUIET)
    if(OpenCV_FOUND)
        list(APPEND PSMOVEAPI_TRACKER_REQUIRED_LIBS ${OpenCV_LIBS})

        set(INFO_BUILD_TRACKER "Yes")
        set(PSMOVEAPI_PKGCONFIG_LIBS "${PSMOVEAPI_PKGCONFIG_LIBS} -lpsmoveapi_tracker")
    else()
        set(INFO_BUILD_TRACKER "No (OpenCV not found)")
        set(PSMOVE_BUILD_TRACKER OFF)
    endif()
else()
    set(INFO_BUILD_TRACKER "No (disabled)")
endif()

set(INFO_USE_CL_EYE_SDK "No (Windows only)")

IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    find_library(IOKIT IOKit)
    find_library(COREFOUNDATION CoreFoundation)
    find_library(IOBLUETOOTH IOBluetooth)
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${IOKIT})
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${COREFOUNDATION})
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${IOBLUETOOTH})

    set(HIDAPI_SRC ${PSMOVEAPI_SOURCE_DIR}/external/hidapi/mac/hid.c)

	# Build Universal Binaries for OS X
	set(CMAKE_OSX_ARCHITECTURES "i386;x86_64")

    list(APPEND PSMOVEAPI_PLATFORM_SRC
        ${PSMOVEAPI_SOURCE_DIR}/src/platform/psmove_osxsupport.m)

    list(APPEND PSMOVEAPI_TRACKER_PLATFORM_SRC
        ${PSMOVEAPI_SOURCE_DIR}/src/tracker/platform/camera_control_macosx.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    list(APPEND PSMOVEAPI_REQUIRED_LIBS setupapi bthprops kernel32 ws2_32 winmm)

    IF(PSMOVE_USE_CL_EYE_SDK)
        find_path(CL_EYE_SDK_PATH Bin/CLEyeMulticam.dll
            HINTS "C:/Program Files/Code Laboratories/CL-Eye Platform SDK"
            "C:/Program Files (x86)/Code Laboratories/CL-Eye Platform SDK")
        IF(CL_EYE_SDK_PATH)
            add_definitions(-DCAMERA_CONTROL_USE_CL_DRIVER)
            list(APPEND PSMOVEAPI_TRACKER_REQUIRED_LIBS CLEyeMulticam)
            file(COPY ${CL_EYE_SDK_PATH}/Bin/CLEyeMulticam.dll
                DESTINATION ${PSMOVEAPI_SOURCE_DIR}/external/libs)
            set(INFO_USE_CL_EYE_SDK "Yes")

            # XXX: If this crashes, disable compiler optimizations
        else()
            set(INFO_USE_CL_EYE_SDK "No (SDK not found)")
        ENDIF()
    else()
        set(INFO_USE_CL_EYE_SDK "No (disabled)")
    ENDIF()

    set(HIDAPI_SRC ${PSMOVEAPI_SOURCE_DIR}/external/hidapi/windows/hid.c)

    list(APPEND PSMOVEAPI_TRACKER_PLATFORM_SRC
        ${PSMOVEAPI_SOURCE_DIR}/src/tracker/platform/camera_control_win32.c)
ELSE()
    # Linux
    find_package(PkgConfig REQUIRED)
    find_package(Threads REQUIRED)
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${CMAKE_THREAD_LIBS_INIT})
    list(APPEND PSMOVEAPI_REQUIRED_LIBS m)

    pkg_check_modules(UDEV REQUIRED libudev)
    include_directories(${UDEV_INCLUDE_DIRS})
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${UDEV_LIBRARIES})

    pkg_check_modules(BLUEZ REQUIRED bluez)
    include_directories(${BLUEZ_INCLUDE_DIRS})
    list(APPEND PSMOVEAPI_REQUIRED_LIBS ${BLUEZ_LIBRARIES})

    set(HIDAPI_SRC ${PSMOVEAPI_SOURCE_DIR}/external/hidapi/linux/hid.c)

    list(APPEND PSMOVEAPI_TRACKER_PLATFORM_SRC
        ${PSMOVEAPI_SOURCE_DIR}/src/tracker/platform/psmove_linuxsupport.c
        ${PSMOVEAPI_SOURCE_DIR}/src/tracker/platform/camera_control_linux.c)
ENDIF()

include_directories(${PSMOVEAPI_SOURCE_DIR}/include)

file(GLOB PSMOVEAPI_MOVED_SRC
    "${PSMOVEAPI_SOURCE_DIR}/src/daemon/*.c"
)

file(GLOB PSMOVEAPI_TRACKER_SRC
    "${PSMOVEAPI_SOURCE_DIR}/src/tracker/*.c"
    "${PSMOVEAPI_SOURCE_DIR}/external/iniparser/*.c"
)

file(GLOB PSMOVEAPI_SRC
    "${PSMOVEAPI_SOURCE_DIR}/src/*.c"
)

file(GLOB PSMOVEAPI_HEADERS
    "${PSMOVEAPI_SOURCE_DIR}/include/*.h"
)

set(PSMOVEAPI_ALGORITHM_SRC
    ${PSMOVEAPI_SOURCE_DIR}/external/MadgwickAHRS/MadgwickAHRS.c
)

# Shared library

add_library(psmoveapi SHARED
    ${HIDAPI_SRC}
    ${PSMOVEAPI_SRC}
    ${PSMOVEAPI_MOVED_SRC}
    ${PSMOVEAPI_PLATFORM_SRC}
    ${PSMOVEAPI_ALGORITHM_SRC})
target_link_libraries(psmoveapi
    ${PSMOVEAPI_REQUIRED_LIBS})
set_target_properties(psmoveapi PROPERTIES
    VERSION ${PSMOVEAPI_VERSION}
    SOVERSION ${PSMOVEAPI_VERSION_SONAME})
set_target_properties(psmoveapi PROPERTIES
    COMPILE_FLAGS -DBUILDING_SHARED_LIBRARY)
list(APPEND PSMOVEAPI_INSTALL_TARGETS psmoveapi)

if(PSMOVE_BUILD_TRACKER)
    add_library(psmoveapi_tracker SHARED
        ${PSMOVEAPI_TRACKER_SRC}
        ${PSMOVEAPI_TRACKER_PLATFORM_SRC})
    target_link_libraries(psmoveapi_tracker
        psmoveapi
        ${PSMOVEAPI_REQUIRED_LIBS}
        ${PSMOVEAPI_TRACKER_REQUIRED_LIBS})
    set_target_properties(psmoveapi_tracker PROPERTIES
        VERSION ${PSMOVEAPI_VERSION}
        SOVERSION ${PSMOVEAPI_VERSION_SONAME})
    set_target_properties(psmoveapi_tracker PROPERTIES
        COMPILE_FLAGS -DBUILDING_SHARED_LIBRARY)
    list(APPEND PSMOVEAPI_INSTALL_TARGETS psmoveapi_tracker)
endif()

# Static library
	add_library(psmoveapi_static STATIC
	    ${HIDAPI_SRC}
	    ${PSMOVEAPI_SRC}
	    ${PSMOVEAPI_MOVED_SRC}
	    ${PSMOVEAPI_PLATFORM_SRC}
	    ${PSMOVEAPI_ALGORITHM_SRC})
	target_link_libraries(psmoveapi_static
	    ${PSMOVEAPI_REQUIRED_LIBS})
	set_target_properties(psmoveapi_static PROPERTIES
	    VERSION ${PSMOVEAPI_VERSION}
	    SOVERSION ${PSMOVEAPI_VERSION_SONAME})

# Essential utilities
foreach(UTILITY moved psmovepair)
    add_executable(${UTILITY} src/utils/${UTILITY}.c)
    target_link_libraries(${UTILITY} psmoveapi)
    list(APPEND PSMOVEAPI_INSTALL_TARGETS ${UTILITY})
endforeach()


# C examples
if(PSMOVE_BUILD_EXAMPLES)
    foreach(EXAMPLE example multiple dump_calibration)
        add_executable(${EXAMPLE} examples/c/${EXAMPLE}.c)
        target_link_libraries(${EXAMPLE} psmoveapi)
    endforeach()

    if(PSMOVE_BUILD_TRACKER AND PSMOVE_BUILD_TUIO_SERVER)
        # You have to run "make -C external/TUIO_CPP static" to build the static
        # library for your platform before the tuio_server can be built here
        include_directories(${PSMOVEAPI_SOURCE_DIR}/external/TUIO_CPP/TUIO)
        include_directories(${PSMOVEAPI_SOURCE_DIR}/external/TUIO_CPP/oscpack)
        link_directories(${PSMOVEAPI_SOURCE_DIR}/external/TUIO_CPP)
        add_executable(tuio_server examples/c/tuio_server.cpp)
        target_link_libraries(tuio_server psmoveapi psmoveapi_tracker TUIO)
    endif()
endif()


# C test programs
if(PSMOVE_BUILD_TESTS)
    foreach(TESTNAME led_update read_performance calibration)
        add_executable(test_${TESTNAME} examples/c/test_${TESTNAME}.c)
        target_link_libraries(test_${TESTNAME} psmoveapi)
    endforeach(TESTNAME)

    if(PSMOVE_BUILD_TRACKER)
        add_executable(test_tracker examples/c/test_tracker.c)
        target_link_libraries(test_tracker psmoveapi psmoveapi_tracker)
    endif()
endif()


# Language bindings (Python, Java and Processing)
find_package(SWIG QUIET)
if(SWIG_FOUND)
    include(${SWIG_USE_FILE})

    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bindings/swig)

    if(PSMOVE_BUILD_PYTHON_BINDINGS)
        find_package(PythonLibs QUIET)
        if(PYTHONLIBS_FOUND)
            unset(CMAKE_SWIG_FLAGS)
            include_directories(${PYTHON_INCLUDE_PATH})
            swig_add_module(psmove python bindings/swig/psmove.i)
            swig_link_libraries(psmove psmoveapi ${PYTHON_LIBRARIES})
            set(INFO_BUILD_PYTHON_BINDINGS "Yes")
        else()
            set(INFO_BUILD_PYTHON_BINDINGS "No (libpython not found)")
        endif()
    else()
        set(INFO_BUILD_PYTHON_BINDINGS "No (disabled)")
    endif()

    if(PSMOVE_BUILD_JAVA_BINDINGS)
        find_package(JNI QUIET)
        if(JNI_FOUND)
            set(PSMOVEAPI_JAVA_PKG "io.thp.psmove")
            string(REPLACE "." "/" PSMOVEAPI_JAVA_PKG_DIR ${PSMOVEAPI_JAVA_PKG})

            set(CMAKE_SWIG_FLAGS -package ${PSMOVEAPI_JAVA_PKG})
            include_directories(${JNI_INCLUDE_DIRS})
            swig_add_module(psmove_java java bindings/swig/psmove.i)
			#swig_link_libraries(psmove_java psmoveapi ${JNI_LIBRARIES})
	        swig_link_libraries(psmove_java psmoveapi_static ${JNI_LIBRARIES})
            set(INFO_BUILD_JAVA_BINDINGS "Yes")

            add_custom_target(psmoveapi.jar ALL
                #COMMAND mkdir -p ${PSMOVEAPI_JAVA_PKG_DIR}
                #COMMAND mv *.java ${PSMOVEAPI_JAVA_PKG_DIR} || true
	            COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_JAVA_PKG_DIR}
				COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/copyjava.cmake ${CMAKE_CURRENT_BINARY_DIR}/copyjava.cmake
	            COMMAND ${CMAKE_COMMAND} -P copyjava.cmake
                COMMAND javac ${PSMOVEAPI_JAVA_PKG_DIR}/*.java
                COMMAND jar cf psmoveapi.jar ${PSMOVEAPI_JAVA_PKG_DIR}/*.class
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                COMMENT "Java bindings JAR archive"
            )
		set(PSMOVEAPI_PROCESSING_DIR "processing/libraries/psmove")
		set(PROCESSING_LIB_DIR "${PSMOVEAPI_PROCESSING_DIR}/library")

		IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
			set(PSMOVEAPI_LIB_PLATFORM "macosx")
			set(PSMOVEAPI_NATIVE_LIB "libpsmove_java.jnilib")
			set(PSMOVEAPI_PAIRING_TOOL_NAME "psmovepair")
		ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
			set(PSMOVEAPI_LIB_PLATFORM "windows")
			set(PSMOVEAPI_NATIVE_LIB "libpsmove_java.dll")
			set(PSMOVEAPI_PAIRING_TOOL_NAME "psmovepair.exe")
		ELSE() # Linux 
			set(PSMOVEAPI_LIB_PLATFORM "linux")
			set(PSMOVEAPI_NATIVE_LIB "libpsmove_java.so")
			set(PSMOVEAPI_PAIRING_TOOL_NAME "psmovepair")
		ENDIF()

		if(PSMOVE_BUILD_PROCESSING_BINDINGS)
		add_custom_target(processing_lib ALL
			# Create Folder Structure
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/examples
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/library
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/library/linux
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/library/macosx
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/library/windows
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/tools
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/tools/linux
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/tools/macosx
			COMMAND ${CMAKE_COMMAND} -E make_directory ${PSMOVEAPI_PROCESSING_DIR}/tools/windows

			# Copy java library, native library, pairing tool, and example app into folder structure
			COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/psmoveapi.jar ${PSMOVEAPI_PROCESSING_DIR}/library/psmove.jar
			COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${PSMOVEAPI_NATIVE_LIB} ${PSMOVEAPI_PROCESSING_DIR}/library/${PSMOVEAPI_LIB_PLATFORM}/${PSMOVEAPI_NATIVE_LIB}
			COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${PSMOVEAPI_PAIRING_TOOL_NAME} ${PSMOVEAPI_PROCESSING_DIR}/tools/${PSMOVEAPI_LIB_PLATFORM}/${PSMOVEAPI_PAIRING_TOOL_NAME}
			COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/processing/libraries/psmove/examples/psmove ${PSMOVEAPI_PROCESSING_DIR}/examples/psmove

			# Cross-platform zipping :) 	
			COMMAND jar cfM psmove_processing.zip processing 

			WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
			COMMENT "Building and packaging Processing library"
			
		)
		
			# Ensure building targets in the correct order 
			# (native lib->java lib->processing lib)
			add_dependencies(psmoveapi.jar psmove_java)
			add_dependencies(processing_lib psmoveapi.jar)
			add_dependencies(processing_lib psmovepair)
		
		set(INFO_BUILD_PROCESSING_BINDINGS "Yes")
		else()
		set(INFO_BUILD_PROCESSING_BINDINGS "No (disabled)")
		endif()
        
        else()
            set(INFO_BUILD_JAVA_BINDINGS "No (JNI not found)")
        endif()
    else()
        set(INFO_BUILD_JAVA_BINDINGS "No (disabled)")
    endif()
else()
    set(INFO_BUILD_PYTHON_BINDINGS "No (SWIG not found)")
    set(INFO_BUILD_JAVA_BINDINGS "No (SWIG not found)")
endif()


# For system-wide installation

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/contrib/psmoveapi.pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/psmoveapi.pc
    @ONLY)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/include/psmove_config.h.in
    ${CMAKE_CURRENT_SOURCE_DIR}/include/psmove_config.h
    @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/psmoveapi.pc
    DESTINATION lib/pkgconfig)

install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/contrib/99-psmove.rules  	
    DESTINATION /etc/udev/rules.d)

install(TARGETS ${PSMOVEAPI_INSTALL_TARGETS}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
)

install(FILES ${PSMOVEAPI_HEADERS} DESTINATION include/psmoveapi)
install(FILES README COPYING LAYOUT INSTALL DESTINATION share/psmoveapi)


# Qt bindings
if(PSMOVE_BUILD_QT_BINDINGS)
    find_package(Qt4 QUIET)

    if(QT_QTDECLARATIVE_FOUND)
        add_subdirectory(bindings/qt)
        set(INFO_BUILD_QT_BINDINGS "Yes")
    else()
        set(INFO_BUILD_QT_BINDINGS "No (Qt not found)")
    endif()
else()
    set(INFO_BUILD_QT_BINDINGS "No (disabled)")
endif()


# Pretty-print if a "use" feature has been enabled
macro(feature_use_info CAPTION FEATURE)
    if(${FEATURE})
        message("    " ${CAPTION} "Yes")
    else()
        message("    " ${CAPTION} "No")
    endif()
endmacro()


# Configuration report
message("")
message("  Build configuration")
message("    Debug build:      " ${INFO_USE_DEBUG})
message("    Tracker library:  " ${INFO_BUILD_TRACKER})
message("")
message("  Language bindings")
message("    Python:           " ${INFO_BUILD_PYTHON_BINDINGS})
message("    Java:             " ${INFO_BUILD_JAVA_BINDINGS})
message("    Processing:       " ${INFO_BUILD_PROCESSING_BINDINGS})
message("    Qt:               " ${INFO_BUILD_QT_BINDINGS})
message("")
message("  Tracker")
feature_use_info("PS Eye support:   " PSMOVE_USE_PSEYE)
feature_use_info("Deinterlacing:    " PSMOVE_USE_DEINTERLACE)
feature_use_info("HTML tracing:     " PSMOVE_USE_TRACKER_TRACE)
message("    Use CL Eye SDK:   " ${INFO_USE_CL_EYE_SDK})
message("")
message("  Additional targets")
feature_use_info("C example apps:   " PSMOVE_BUILD_EXAMPLES)
feature_use_info("C test programs:  " PSMOVE_BUILD_TESTS)
feature_use_info("C++ TUIO server:  " PSMOVE_BUILD_TUIO_SERVER)
message("")

