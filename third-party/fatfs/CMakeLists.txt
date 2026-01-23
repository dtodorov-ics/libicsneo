project(fatfs VERSION 0.14.1) # R0.14b

add_library(fatfs
	source/ff.c
	source/ffunicode.c
	source/ffsystem.c
)
target_include_directories(fatfs 
	PUBLIC 
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/source>
		$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/fatfs>
)

# Install fatfs headers for vcpkg
install(FILES
    source/ff.h
    source/ffconf.h
    source/diskio.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/fatfs
)
