cmake_minimum_required(VERSION 3.16)
project(libicsneo VERSION 1.0.0)

cmake_policy(SET CMP0074 NEW)
if(POLICY CMP0135)
  cmake_policy(SET CMP0135 NEW)
endif()

option(LIBICSNEO_BUILD_UNIT_TESTS "Build unit tests." OFF)
option(LIBICSNEO_BUILD_DOCS "Build documentation. Don't use in Visual Studio." OFF)
option(LIBICSNEO_BUILD_EXAMPLES "Build examples." ON)
option(LIBICSNEO_BUILD_ICSNEOC "Build dynamic C library" ON)
option(LIBICSNEO_BUILD_ICSNEOC_STATIC "Build static C library" ON)
option(LIBICSNEO_BUILD_ICSNEOLEGACY "Build icsnVC40 compatibility library" ON)
option(LIBICSNEO_BUILD_ICSNEOLEGACY_STATIC "Build static icsnVC40 compatibility library" ON)
set(LIBICSNEO_NPCAP_INCLUDE_DIR "" CACHE STRING "Npcap include directory; set to build with Npcap")

# Device Drivers
# You almost certainly don't want firmio for your build,
# it is only relevant for communication between Linux and
# CoreMini from the onboard processor of the device.
option(LIBICSNEO_ENABLE_FIRMIO "Enable communication between Linux and CoreMini within the same device" OFF)
option(LIBICSNEO_ENABLE_RAW_ETHERNET "Enable devices which communicate over raw ethernet" ON)
option(LIBICSNEO_ENABLE_CDCACM "Enable devices which communicate over USB CDC ACM" ON)
option(LIBICSNEO_ENABLE_TCP "Enable devices which communicate over TCP" OFF)
option(LIBICSNEO_ENABLE_DXX "Enable devices which communicate over D2XX/D3XX via libredxx" ON)

option(LIBICSNEO_ENABLE_BINDINGS_PYTHON "Enable Python library" OFF)

set(CMAKE_CXX_STANDARD 17)

include(GNUInstallDirs)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Detect vcpkg toolchain
if(DEFINED CMAKE_TOOLCHAIN_FILE AND CMAKE_TOOLCHAIN_FILE MATCHES "vcpkg")
	set(USE_VCPKG ON)
	message(STATUS "[libicsneo] vcpkg toolchain detected")
endif()

# Set MSVC runtime library only when not using vcpkg (vcpkg controls this via triplet)
if(MSVC AND NOT USE_VCPKG)
	set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# Enable Warnings
if(MSVC)
	set(LIBICSNEO_COMPILER_WARNINGS /W4)
	# http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0618r0.html
	# Still supported until a suitable replacement is standardized
	add_definitions(-D_SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING)
	add_definitions(-D_ITERATOR_DEBUG_LEVEL=0)
else() #if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
	set(LIBICSNEO_COMPILER_WARNINGS -Wall -Wno-switch -Wno-unknown-pragmas)
endif()

find_package(Threads REQUIRED)

# doxygen
find_package(Doxygen)
if(DOXYGEN_FOUND)
	set(DOXYGEN_OUT ${CMAKE_CURRENT_SOURCE_DIR}/docs/Doxyfile)
	set(ICSNEO_DOCS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/docs)
	if(NOT EXISTS "${DOXYGEN_OUT}")
		set(DOXYGEN_FOUND FALSE)
	endif()
endif()

if(LIBICSNEO_BUILD_DOCS)
	if(DOXYGEN_FOUND)
		message("Will build Doxygen based documentation")

		add_custom_target(libicsneo_doxygen
			COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
			WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
			COMMENT "Generating API documentation with Doxygen"
			VERBATIM
			DEPENDS icsneocpp icsneoc icsneolegacy)

		# sphinx
		find_package(Sphinx)
		if(SPHINX_EXECUTABLE)
			message("Will build Sphinx based documentation")

			set(SPHINX_OUT ${ICSNEO_DOCS_DIR}/build/sphinx)

			# configured documentation tools and intermediate build results
			set(SPHINX_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR}/_build)

			# Sphinx cache with pickled ReST documents
			set(SPHINX_CACHE_DIR ${CMAKE_CURRENT_BINARY_DIR}/_doctrees)

			# HTML output directory
			set(SPHINX_HTML_DIR ${CMAKE_CURRENT_BINARY_DIR}/doc_sphinx)

			configure_file(
				"${ICSNEO_DOCS_DIR}/conf.py.template"
				"${ICSNEO_DOCS_DIR}/conf.py"
				@ONLY)

			add_custom_target(libicsneo_sphinx ALL
				${SPHINX_EXECUTABLE}
					-q -b html
					-c "${ICSNEO_DOCS_DIR}"
					-d "${SPHINX_CACHE_DIR}"
					"${ICSNEO_DOCS_DIR}"
					"${SPHINX_HTML_DIR}"
				COMMENT "Building HTML documentation with Sphinx"
				DEPENDS icsneocpp icsneoc icsneolegacy)
		endif()
	endif()
endif()

if(WIN32)
	set(PLATFORM_SRC
		platform/windows/strings.cpp
		platform/windows/registry.cpp
	)

	if(LIBICSNEO_ENABLE_RAW_ETHERNET)
		list(APPEND PLATFORM_SRC
			platform/windows/pcap.cpp
			platform/windows/internal/pcapdll.cpp
		)
	endif()

	if(LIBICSNEO_ENABLE_CDCACM)
		list(APPEND PLATFORM_SRC
			platform/windows/cdcacm.cpp
		)
	endif()
else() # Darwin or Linux
	set(PLATFORM_SRC)

	if(LIBICSNEO_ENABLE_FIRMIO)
		list(APPEND PLATFORM_SRC
			platform/posix/firmio.cpp
		)
	endif()

	if(LIBICSNEO_ENABLE_RAW_ETHERNET)
		list(APPEND PLATFORM_SRC
			platform/posix/pcap.cpp
		)
	endif()

	if(LIBICSNEO_ENABLE_CDCACM)
		list(APPEND PLATFORM_SRC
			platform/posix/cdcacm.cpp
		)

		if(${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")
			list(APPEND PLATFORM_SRC
				platform/posix/darwin/cdcacmdarwin.cpp
			)
		else() # Linux or other
			list(APPEND PLATFORM_SRC
				platform/posix/linux/cdcacmlinux.cpp
			)
			if(NOT ${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
				message(WARNING
					"There is no CDCACM platform port defined for ${CMAKE_SYSTEM_NAME}!\n"
					"The Linux platform code will be used, as it will generally allow building, but some devices may not enumerate properly."
				)
			endif()
		endif()
	endif()
endif()

if(LIBICSNEO_ENABLE_DXX)
	list(APPEND PLATFORM_SRC
		platform/dxx.cpp
	)
endif()

if(LIBICSNEO_ENABLE_TCP)
	list(APPEND PLATFORM_SRC
		platform/tcp.cpp
	)
endif()

if(LIBICSNEO_BUILD_EXAMPLES)
	add_subdirectory(examples)
endif()

# Extensions
set(LIBICSNEO_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
foreach(EXT_PATH ${LIBICSNEO_EXTENSION_DIRS})
	get_filename_component(EXT_DIR ${EXT_PATH} NAME)
	message("Adding extension " ${EXT_DIR})
	add_subdirectory(${EXT_PATH} ${CMAKE_CURRENT_BINARY_DIR}/${EXT_DIR})
endforeach()

set(SRC_FILES
	communication/message/flexray/control/flexraycontrolmessage.cpp
	communication/message/callback/streamoutput/a2bwavoutput.cpp
	communication/message/a2bmessage.cpp
	communication/message/apperrormessage.cpp
	communication/message/neomessage.cpp
	communication/message/ethphymessage.cpp
	communication/message/linmessage.cpp
	communication/message/livedatamessage.cpp
	communication/message/logdatamessage.cpp
	communication/message/tc10statusmessage.cpp
	communication/message/gptpstatusmessage.cpp
	communication/message/ethernetstatusmessage.cpp
	communication/message/networkmutexmessage.cpp
	communication/message/clientidmessage.cpp
	communication/message/transmitmessage.cpp
	communication/packet/flexraypacket.cpp
	communication/packet/canpacket.cpp
	communication/packet/a2bpacket.cpp
	communication/packet/spipacket.cpp
	communication/packet/ethernetpacket.cpp
	communication/packet/versionpacket.cpp
	communication/packet/iso9141packet.cpp
	communication/packet/ethphyregpacket.cpp
	communication/packet/livedatapacket.cpp
	communication/packet/logicaldiskinfopacket.cpp
	communication/packet/wivicommandpacket.cpp
	communication/packet/i2cpacket.cpp
	communication/packet/linpacket.cpp
	communication/packet/mdiopacket.cpp
	communication/packet/scriptstatuspacket.cpp
	communication/packet/componentversionpacket.cpp
	communication/packet/supportedfeaturespacket.cpp
	communication/packet/genericbinarystatuspacket.cpp
	communication/packet/hardwareinfopacket.cpp
	communication/decoder.cpp
	communication/encoder.cpp
	communication/ethernetpacketizer.cpp
	communication/packetizer.cpp
	communication/multichannelcommunication.cpp
	communication/communication.cpp
	communication/driver.cpp
	communication/livedata.cpp
	core/ringbuffer.cpp
	core/crc32.cpp
	core/macseccfg.cpp
	device/extensions/flexray/extension.cpp
	device/extensions/flexray/controller.cpp
	device/idevicesettings.cpp
	device/devicefinder.cpp
	device/device.cpp
	device/neodevice.cpp
	disk/diskreaddriver.cpp
	disk/diskwritedriver.cpp
	disk/nulldiskdriver.cpp
	disk/neomemorydiskdriver.cpp
	disk/plasiondiskreaddriver.cpp
	disk/extextractordiskreaddriver.cpp
	disk/fat.cpp
	disk/diskdetails.cpp
	disk/vsa/vsa.cpp
	disk/vsa/vsa02.cpp
	disk/vsa/vsa03.cpp
	disk/vsa/vsa04.cpp
	disk/vsa/vsa05.cpp
	disk/vsa/vsa06.cpp
	disk/vsa/vsa07.cpp
	disk/vsa/vsa08.cpp
	disk/vsa/vsa09.cpp
	disk/vsa/vsa0b.cpp
	disk/vsa/vsa0c.cpp
	disk/vsa/vsa0d.cpp
	disk/vsa/vsa0e.cpp
	disk/vsa/vsa0f.cpp
	disk/vsa/vsa6a.cpp
	disk/vsa/vsaparser.cpp
	platform/servd.cpp
	${PLATFORM_SRC}
)

# Generate build info header
execute_process(
	COMMAND git rev-parse --abbrev-ref HEAD
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	OUTPUT_VARIABLE GIT_BRANCH
	OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
	COMMAND git describe --abbrev=6 --dirty --always --tags
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	OUTPUT_VARIABLE GIT_DESCRIBE
	ERROR_VARIABLE GIT_DESCRIBE
	OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(${BUILD_METADATA})
	set(BUILD_METADATA_PLUS +${BUILD_METADATA})
endif()
if(NOT ${GIT_BRANCH} STREQUAL "master")
	set(BUILD_GIT_INFO "${GIT_BRANCH} @ ")
endif()
string(SUBSTRING GIT_DESCRIBE 0 1 GIT_DESCRIBE_FIRST)
if(NOT ${GIT_DESCRIBE_FIRST} STREQUAL "v")
	string(APPEND BUILD_GIT_INFO "${GIT_DESCRIBE}")
endif()

configure_file(api/icsneocpp/buildinfo.h.template ${CMAKE_CURRENT_BINARY_DIR}/generated/buildinfo.h)
configure_file(api/icsneoc/version.rc.template ${CMAKE_CURRENT_BINARY_DIR}/generated/icsneoc/version.rc)

foreach(EXTINC ${LIBICSNEO_EXTENSION_INCLUDES})
	message("Including " ${EXTINC})
	list(APPEND LIBICSNEO_EXT_CODE_INCS_LIST "#include \"${EXTINC}\"")
endforeach()
list(JOIN LIBICSNEO_EXT_CODE_INCS_LIST "\n" LIBICSNEO_EXT_CODE_INCS)
foreach(EXTCLASS ${LIBICSNEO_EXTENSION_CLASSES})
	list(APPEND LIBICSNEO_EXT_CODE_LIST "device->addExtension(std::make_shared<${EXTCLASS}>(*device))\;")
endforeach()
list(JOIN LIBICSNEO_EXT_CODE_LIST "\n\t" LIBICSNEO_EXT_CODE)
configure_file(include/icsneo/device/extensions/builtin.h.template ${CMAKE_CURRENT_BINARY_DIR}/generated/extensions/builtin.h)

include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR})

add_library(icsneocpp
	api/icsneocpp/icsneocpp.cpp
	api/icsneocpp/event.cpp
	api/icsneocpp/eventmanager.cpp
	api/icsneocpp/version.cpp
	${SRC_FILES}
)
message("Include paths " ${LIBICSNEO_EXTENSION_INCLUDE_PATHS})
target_include_directories(icsneocpp
	PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
		$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
	PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/include
		${LIBICSNEO_EXTENSION_INCLUDE_PATHS}
)
target_link_libraries(icsneocpp PUBLIC Threads::Threads $<$<BOOL:${WIN32}>:ws2_32 iphlpapi>)
set_property(TARGET icsneocpp PROPERTY POSITION_INDEPENDENT_CODE ON)
target_compile_features(icsneocpp PUBLIC cxx_auto_type cxx_constexpr cxx_lambdas cxx_nullptr cxx_range_for cxx_rvalue_references cxx_sizeof_member cxx_strong_enums)
target_compile_options(icsneocpp PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})
message("Loaded extensions: " ${LIBICSNEO_EXTENSION_TARGETS})
target_link_libraries(icsneocpp PUBLIC ${LIBICSNEO_EXTENSION_TARGETS})
if(LIBICSNEO_ENABLE_FIRMIO)
	target_compile_definitions(icsneocpp PRIVATE ICSNEO_ENABLE_FIRMIO)
endif()
if(LIBICSNEO_ENABLE_RAW_ETHERNET)
	target_compile_definitions(icsneocpp PRIVATE ICSNEO_ENABLE_RAW_ETHERNET)
endif()
if(LIBICSNEO_ENABLE_CDCACM)
	target_compile_definitions(icsneocpp PRIVATE ICSNEO_ENABLE_CDCACM)
	if(APPLE)
		find_library(CORE_FOUNDATION_FRAMEWORK CoreFoundation REQUIRED)
		find_library(IO_KIT_FRAMEWORK IOKit REQUIRED)
		target_link_libraries(icsneocpp PRIVATE ${CORE_FOUNDATION_FRAMEWORK} ${IO_KIT_FRAMEWORK})
	endif()
	# libusb for CDCACM support (all platforms including Windows)
	set(LIBUSB_FOUND FALSE)
	
	# Try pkg-config first (works with vcpkg or system pkg-config)
	find_package(PkgConfig QUIET)
	if(PKG_CONFIG_FOUND)
		pkg_check_modules(LIBUSB IMPORTED_TARGET libusb-1.0)
		if(LIBUSB_FOUND)
			target_link_libraries(icsneocpp PRIVATE $<BUILD_INTERFACE:PkgConfig::LIBUSB>)
			message(STATUS "[libicsneo] Using libusb via pkg-config")
		endif()
	endif()
	
	# Fallback: Try find_package for libusb (CMake config mode)
	if(NOT LIBUSB_FOUND)
		find_package(libusb CONFIG QUIET)
		if(libusb_FOUND)
			target_link_libraries(icsneocpp PRIVATE libusb::libusb)
			set(LIBUSB_FOUND TRUE)
			message(STATUS "[libicsneo] Using libusb via CMake find_package")
		endif()
	endif()
	
	# Fallback for Windows: manual path specification via LIBUSB_ROOT
	if(NOT LIBUSB_FOUND AND WIN32)
		if(DEFINED LIBUSB_ROOT OR DEFINED ENV{LIBUSB_ROOT})
			if(DEFINED ENV{LIBUSB_ROOT} AND NOT DEFINED LIBUSB_ROOT)
				set(LIBUSB_ROOT "$ENV{LIBUSB_ROOT}")
			endif()
			message(STATUS "[libicsneo] Looking for libusb in LIBUSB_ROOT: ${LIBUSB_ROOT}")
			find_path(LIBUSB_INCLUDE_DIR libusb.h
				PATHS "${LIBUSB_ROOT}/include" "${LIBUSB_ROOT}/include/libusb-1.0"
				NO_DEFAULT_PATH)
			find_library(LIBUSB_LIBRARY
				NAMES libusb-1.0 usb-1.0 libusb
				PATHS "${LIBUSB_ROOT}/lib" 
					"${LIBUSB_ROOT}/VS2022/MS64/static" "${LIBUSB_ROOT}/VS2022/MS64/dll"
					"${LIBUSB_ROOT}/VS2019/MS64/static" "${LIBUSB_ROOT}/VS2019/MS64/dll"
				NO_DEFAULT_PATH)
			if(LIBUSB_INCLUDE_DIR AND LIBUSB_LIBRARY)
				target_include_directories(icsneocpp PRIVATE ${LIBUSB_INCLUDE_DIR})
				target_link_libraries(icsneocpp PRIVATE ${LIBUSB_LIBRARY})
				set(LIBUSB_FOUND TRUE)
				message(STATUS "[libicsneo] Using libusb from LIBUSB_ROOT: ${LIBUSB_ROOT}")
				message(STATUS "[libicsneo]   Include: ${LIBUSB_INCLUDE_DIR}")
				message(STATUS "[libicsneo]   Library: ${LIBUSB_LIBRARY}")
			endif()
		endif()
	endif()
	
	# Fallback for Linux/macOS: system paths
	if(NOT LIBUSB_FOUND AND NOT WIN32)
		find_path(LIBUSB_INCLUDE_DIR libusb.h PATH_SUFFIXES libusb-1.0)
		find_library(LIBUSB_LIBRARY NAMES usb-1.0 libusb-1.0)
		if(LIBUSB_INCLUDE_DIR AND LIBUSB_LIBRARY)
			target_include_directories(icsneocpp PRIVATE ${LIBUSB_INCLUDE_DIR})
			target_link_libraries(icsneocpp PRIVATE ${LIBUSB_LIBRARY})
			set(LIBUSB_FOUND TRUE)
			message(STATUS "[libicsneo] Using system libusb")
		endif()
	endif()
	
	# Final fallback: Download pre-built libusb binaries via FetchContent
	if(NOT LIBUSB_FOUND)
		message(STATUS "[libicsneo] libusb not found - downloading pre-built binaries via FetchContent")
		include(FetchContent)
		
		set(LIBUSB_VERSION "1.0.27")
		
		if(WIN32)
			# Download pre-built Windows binaries (much simpler than building from source)
			set(LIBUSB_URL "https://github.com/libusb/libusb/releases/download/v${LIBUSB_VERSION}/libusb-${LIBUSB_VERSION}.7z")
			set(LIBUSB_HASH "SHA256=19835e290f46fab6bd8ce4be6ab7dc5209f1c04bad177065df485e51dc4118c8")
			
			FetchContent_Declare(libusb_bin
				URL ${LIBUSB_URL}
				URL_HASH ${LIBUSB_HASH}
				DOWNLOAD_EXTRACT_TIMESTAMP TRUE
			)
			FetchContent_MakeAvailable(libusb_bin)
			
			set(LIBUSB_ROOT "${libusb_bin_SOURCE_DIR}")
			set(LIBUSB_INCLUDE_DIR "${LIBUSB_ROOT}/include/libusb-1.0")
			
			# Determine architecture for library path
			if(CMAKE_SIZEOF_VOID_P EQUAL 8)
				set(LIBUSB_ARCH "x64")
			else()
				set(LIBUSB_ARCH "Win32")
			endif()
			
			# Check for VS2019+ static lib first, then VS2015
			if(EXISTS "${LIBUSB_ROOT}/VS2019/MS64/static/libusb-1.0.lib")
				set(LIBUSB_LIB_DIR "${LIBUSB_ROOT}/VS2019/MS64/static")
			elseif(EXISTS "${LIBUSB_ROOT}/VS2015-${LIBUSB_ARCH}/static/libusb-1.0.lib")
				set(LIBUSB_LIB_DIR "${LIBUSB_ROOT}/VS2015-${LIBUSB_ARCH}/static")
			else()
				# Generic path fallback
				set(LIBUSB_LIB_DIR "${LIBUSB_ROOT}/VS2019/MS64/static")
			endif()
			
			find_library(LIBUSB_LIBRARY NAMES libusb-1.0 PATHS "${LIBUSB_LIB_DIR}" NO_DEFAULT_PATH)
			
			if(LIBUSB_LIBRARY)
				target_include_directories(icsneocpp PRIVATE ${LIBUSB_INCLUDE_DIR})
				target_link_libraries(icsneocpp PRIVATE ${LIBUSB_LIBRARY})
				set(LIBUSB_FOUND TRUE)
				message(STATUS "[libicsneo] Using downloaded pre-built libusb (v${LIBUSB_VERSION})")
			else()
				message(WARNING "[libicsneo] Downloaded libusb but could not find library at ${LIBUSB_LIB_DIR}")
			endif()
		else()
			# For non-Windows, download source and try to build (simpler on Unix)
			# Or require system installation
			message(FATAL_ERROR "[libicsneo] CDCACM is enabled but libusb-1.0 could not be found.\n"
				"Please install libusb-1.0 development package:\n"
				"  Ubuntu/Debian: sudo apt-get install libusb-1.0-0-dev\n"
				"  Fedora: sudo dnf install libusb1-devel\n"
				"  macOS: brew install libusb")
		endif()
	endif()
	
	if(NOT LIBUSB_FOUND)
		message(FATAL_ERROR "[libicsneo] CDCACM is enabled but libusb-1.0 could not be found or built.\n"
			"This should not happen - please report this issue.")
	endif()
endif()
if(LIBICSNEO_ENABLE_DXX)
	target_compile_definitions(icsneocpp PRIVATE ICSNEO_ENABLE_DXX)
	target_link_libraries(icsneocpp PRIVATE $<BUILD_INTERFACE:libredxx::libredxx>)
endif()
if(LIBICSNEO_ENABLE_TCP)
	target_compile_definitions(icsneocpp PRIVATE ICSNEO_ENABLE_TCP)
	if(WIN32)
		target_link_libraries(icsneocpp PRIVATE ws2_32 iphlpapi)
	endif()
endif()

# fatfs
add_subdirectory(third-party/fatfs)
set_property(TARGET fatfs PROPERTY POSITION_INDEPENDENT_CODE ON)
target_link_libraries(icsneocpp PRIVATE $<BUILD_INTERFACE:fatfs>)

# dxx
if(LIBICSNEO_ENABLE_DXX)
	include(FetchContent)
	FetchContent_Declare(libredxx
		GIT_REPOSITORY https://github.com/Zeranoe/libredxx.git
		GIT_TAG 267abf26a99fa69ed80a4180b155245a36fad101
	)
	# Don't disable install - vcpkg needs libredxx to be installed
	# set(LIBREDXX_DISABLE_INSTALL ON)
	FetchContent_MakeAvailable(libredxx)
endif()

# pcap
if(LIBICSNEO_ENABLE_RAW_ETHERNET)
	if(WIN32)
		if(LIBICSNEO_NPCAP_INCLUDE_DIR STREQUAL "")
			target_include_directories(icsneocpp PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third-party/winpcap/include>)
			add_definitions(-DWPCAP -DHAVE_REMOTE)
		else()
			target_include_directories(icsneocpp PUBLIC AFTER ${LIBICSNEO_NPCAP_INCLUDE_DIR})
		endif()
	else()
		find_package(PCAP REQUIRED)
		target_include_directories(icsneocpp PUBLIC ${PCAP_INCLUDE_DIR})
		target_link_libraries(icsneocpp PUBLIC ${PCAP_LIBRARY})
	endif(WIN32)
endif(LIBICSNEO_ENABLE_RAW_ETHERNET)

# protobuf / icspb configuration
# icspb requires protobuf. If protobuf_ROOT is set, icspb will use it.
# Otherwise, icspb will try to bootstrap protobuf.
# The bootstrap can fail on some systems (especially Windows with long paths).
# We try to find system protobuf first to avoid the bootstrap.
if(USE_VCPKG)
	# When using vcpkg, set protobuf_ROOT to use vcpkg's protobuf and skip icspb's bootstrap
	if(NOT DEFINED protobuf_ROOT)
		set(protobuf_ROOT "${CMAKE_CURRENT_BINARY_DIR}/vcpkg_installed/${VCPKG_TARGET_TRIPLET}")
		message(STATUS "[libicsneo] Setting protobuf_ROOT for icspb: ${protobuf_ROOT}")
	endif()
elseif(NOT DEFINED protobuf_ROOT)
	# Not using vcpkg and protobuf_ROOT not set - try to find system protobuf
	find_package(protobuf CONFIG QUIET)
	if(protobuf_FOUND)
		message(STATUS "[libicsneo] Found system protobuf: ${protobuf_CONFIG}")
		# Extract install root from config path by walking up 3 directory levels
		get_filename_component(protobuf_ROOT "${protobuf_CONFIG}" DIRECTORY)
		get_filename_component(protobuf_ROOT "${protobuf_ROOT}" DIRECTORY)
		get_filename_component(protobuf_ROOT "${protobuf_ROOT}" DIRECTORY)
		message(STATUS "[libicsneo] Setting protobuf_ROOT for icspb: ${protobuf_ROOT}")
	else()
		# System protobuf not found - icspb will attempt to bootstrap it
		message(STATUS "[libicsneo] System protobuf not found - icspb will bootstrap protobuf")
		message(STATUS "[libicsneo] To use pre-installed protobuf, set -Dprotobuf_ROOT=<path>")
		if(WIN32)
			message(STATUS "[libicsneo] On Windows, consider using vcpkg: cmake -DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake")
		endif()
	endif()
endif()
if(DEFINED ENV{LIBICSNEO_ICSPB_REPO})
	set(LIBICSNEO_ICSPB_REPO "$ENV{LIBICSNEO_ICSPB_REPO}")
endif()
if(NOT LIBICSNEO_ICSPB_REPO)
	set(LIBICSNEO_ICSPB_REPO https://github.com/intrepidcs/icspb.git)
endif()
if(DEFINED ENV{LIBICSNEO_ICSPB_TAG})
	set(LIBICSNEO_ICSPB_TAG "$ENV{LIBICSNEO_ICSPB_TAG}")
endif()
if(NOT LIBICSNEO_ICSPB_TAG)
	set(LIBICSNEO_ICSPB_TAG 3339fa6b83a6b3e7704d41f5c2f2175cfc761a1f)
endif()
include(FetchContent)
FetchContent_Declare(icspb
	GIT_REPOSITORY ${LIBICSNEO_ICSPB_REPO}
	GIT_TAG ${LIBICSNEO_ICSPB_TAG}
)

# On Windows, use a shorter bootstrap path to avoid path length issues
# The default path can exceed MAX_PATH (260 chars) causing absl build failures
if(WIN32 AND NOT DEFINED ICSPB_BOOTSTRAP_DIR)
	set(ICSPB_BOOTSTRAP_DIR "${CMAKE_SOURCE_DIR}/_pb_boot" CACHE PATH "icspb protobuf bootstrap directory")
	message(STATUS "[libicsneo] Setting ICSPB_BOOTSTRAP_DIR to short path: ${ICSPB_BOOTSTRAP_DIR}")
endif()

FetchContent_MakeAvailable(icspb)

target_link_libraries(icsneocpp PRIVATE $<BUILD_INTERFACE:icspb::icspb>)

if(LIBICSNEO_BUILD_ICSNEOC)
	add_library(icsneoc SHARED api/icsneoc/icsneoc.cpp ${CMAKE_CURRENT_BINARY_DIR}/generated/icsneoc/version.rc)
	target_include_directories(icsneoc
		PUBLIC
			$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
			$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		PRIVATE
			${CMAKE_CURRENT_SOURCE_DIR}/include
	)
	target_link_libraries(icsneoc PRIVATE icsneocpp)
	target_compile_features(icsneoc PRIVATE cxx_auto_type cxx_constexpr cxx_lambdas cxx_nullptr cxx_range_for cxx_rvalue_references cxx_sizeof_member cxx_strong_enums)
	target_compile_options(icsneoc PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})
endif()

if(LIBICSNEO_BUILD_ICSNEOC_STATIC)
	add_library(icsneoc-static STATIC api/icsneoc/icsneoc.cpp)
	target_include_directories(icsneoc-static
		PUBLIC
			$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
			$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		PRIVATE
			${CMAKE_CURRENT_SOURCE_DIR}/include
	)
	target_link_libraries(icsneoc-static PUBLIC icsneocpp)
	target_compile_features(icsneoc-static PUBLIC cxx_auto_type cxx_constexpr cxx_lambdas cxx_nullptr cxx_range_for cxx_rvalue_references cxx_sizeof_member cxx_strong_enums)
	target_compile_definitions(icsneoc-static PUBLIC ICSNEOC_BUILD_STATIC)
	target_compile_options(icsneoc-static PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})
endif()

if(LIBICSNEO_BUILD_ICSNEOLEGACY)
	add_library(icsneolegacy SHARED
		api/icsneolegacy/icsneolegacy.cpp
		api/icsneolegacy/icsneolegacyextra.cpp
		api/icsneoc/icsneoc.cpp
		platform/windows/icsneolegacy.def
	)
	target_include_directories(icsneolegacy
		PUBLIC
			$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
			$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		PRIVATE
			${CMAKE_CURRENT_SOURCE_DIR}/include
	)
	target_link_libraries(icsneolegacy PRIVATE icsneocpp)
	target_compile_features(icsneolegacy PRIVATE cxx_auto_type cxx_constexpr cxx_lambdas cxx_nullptr cxx_range_for cxx_rvalue_references cxx_sizeof_member cxx_strong_enums)
	target_compile_options(icsneolegacy PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})
endif()

if(LIBICSNEO_BUILD_ICSNEOLEGACY_STATIC)
	add_library(icsneolegacy-static STATIC
		api/icsneolegacy/icsneolegacy.cpp
		api/icsneolegacy/icsneolegacyextra.cpp
		api/icsneoc/icsneoc.cpp
	)

	target_include_directories(icsneolegacy-static
		PUBLIC
			$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
			$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		PRIVATE
			${CMAKE_CURRENT_SOURCE_DIR}/include
	)
	target_link_libraries(icsneolegacy-static PUBLIC icsneocpp)
	target_compile_features(icsneolegacy-static PUBLIC cxx_auto_type cxx_constexpr cxx_lambdas cxx_nullptr cxx_range_for cxx_rvalue_references cxx_sizeof_member cxx_strong_enums)
	target_compile_definitions(icsneolegacy-static PUBLIC ICSNEOC_BUILD_STATIC)
	target_compile_options(icsneolegacy-static PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})
endif()

add_subdirectory(bindings)

# googletest
if(LIBICSNEO_BUILD_UNIT_TESTS)
	include(FetchContent)
	FetchContent_Declare(googletest
		GIT_REPOSITORY https://github.com/google/googletest.git
		GIT_TAG 6986c2b575f77135401a4e1c65a7a42f20e18fef
	)
	FetchContent_MakeAvailable(googletest)

	add_executable(libicsneo-unit-tests
		test/unit/main.cpp
		test/unit/diskdriverreadtest.cpp
		test/unit/diskdriverwritetest.cpp
		test/unit/eventmanagertest.cpp
		test/unit/ethernetpacketizertest.cpp
		test/unit/i2cencoderdecodertest.cpp
		test/unit/linencoderdecodertest.cpp
		test/unit/a2bencoderdecodertest.cpp
		test/unit/mdioencoderdecodertest.cpp
		test/unit/livedataencoderdecodertest.cpp
		test/unit/ringbuffertest.cpp
		test/unit/apperrordecodertest.cpp
		test/unit/windowsstrings.cpp
		test/unit/periodictest.cpp
	)

	target_compile_options(libicsneo-unit-tests PRIVATE ${LIBICSNEO_COMPILER_WARNINGS})

	target_link_libraries(libicsneo-unit-tests gtest gtest_main)
	target_link_libraries(libicsneo-unit-tests icsneocpp)

	target_include_directories(libicsneo-unit-tests PUBLIC ${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})

	enable_testing()

	add_test(NAME libicsneo-unit-test-suite COMMAND libicsneo-unit-tests)
endif()


# ============================================================================
# vcpkg Installation Support
# ============================================================================

include(CMakePackageConfigHelpers)

# Install targets
set(LIBICSNEO_INSTALL_TARGETS)

# Always install icsneocpp (core library)
list(APPEND LIBICSNEO_INSTALL_TARGETS icsneocpp)

# Conditionally install C API libraries
if(LIBICSNEO_BUILD_ICSNEOC)
    list(APPEND LIBICSNEO_INSTALL_TARGETS icsneoc)
endif()

if(LIBICSNEO_BUILD_ICSNEOC_STATIC)
    list(APPEND LIBICSNEO_INSTALL_TARGETS icsneoc-static)
endif()

# Conditionally install legacy API libraries
if(LIBICSNEO_BUILD_ICSNEOLEGACY)
    list(APPEND LIBICSNEO_INSTALL_TARGETS icsneolegacy)
endif()

if(LIBICSNEO_BUILD_ICSNEOLEGACY_STATIC)
    list(APPEND LIBICSNEO_INSTALL_TARGETS icsneolegacy-static)
endif()

# Install bundled/FetchContent dependencies for package managers
# These are normally BUILD_INTERFACE only, but vcpkg needs them installed
list(APPEND LIBICSNEO_INSTALL_TARGETS fatfs)
if(LIBICSNEO_ENABLE_DXX AND TARGET libredxx)
    list(APPEND LIBICSNEO_INSTALL_TARGETS libredxx)
endif()

# Install all configured targets
install(TARGETS ${LIBICSNEO_INSTALL_TARGETS}
    EXPORT libicsneo-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Install public headers
install(DIRECTORY include/icsneo
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h"
    PATTERN "*.template" EXCLUDE
)

# Install generated headers
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/generated/buildinfo.h"
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/icsneo
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/generated/extensions/builtin.h"
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/icsneo/device/extensions
)

# Export targets
install(EXPORT libicsneo-targets
    FILE libicsneo-targets.cmake
    NAMESPACE libicsneo::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libicsneo
)

# Generate config file
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/libicsneo-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/libicsneo-config.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libicsneo
)

# Generate version file
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/libicsneo-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# Install config files
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/libicsneo-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/libicsneo-config-version.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libicsneo
)

# Install pkg-config file (optional but recommended)
if(UNIX)
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/libicsneo.pc.in"
        "${CMAKE_CURRENT_BINARY_DIR}/libicsneo.pc"
        @ONLY
    )
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libicsneo.pc"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()
set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
include(CPack)
