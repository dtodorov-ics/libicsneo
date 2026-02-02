=================
vcpkg Installation
=================

libicsneo can be installed via `vcpkg <https://vcpkg.io/>`_ using an overlay port.

Prerequisites
=============

- A vcpkg-managed CMake project
- vcpkg installed and bootstrapped
- On Linux: ``libpcap-dev`` and ``pkg-config`` system packages

Overlay Port Setup
==================

Create the following files in your project's ``ports/libicsneo/`` directory:

``ports/libicsneo/vcpkg.json``
------------------------------

.. code-block:: json

    {
      "name": "libicsneo",
      "version": "1.0.1",
      "description": "Library for communicating with Intrepid Control Systems hardware",
      "homepage": "https://github.com/intrepidcs/libicsneo",
      "license": "MIT",
      "supports": "!uwp",
      "dependencies": [
        "libusb",
        "protobuf",
        { "name": "libpcap", "platform": "!windows" },
        { "name": "pkgconf", "host": true },
        { "name": "vcpkg-cmake", "host": true },
        { "name": "vcpkg-cmake-config", "host": true }
      ]
    }

``ports/libicsneo/portfile.cmake``
----------------------------------

.. code-block:: cmake

    vcpkg_check_linkage(ONLY_STATIC_LIBRARY)

    vcpkg_from_github(
        OUT_SOURCE_PATH SOURCE_PATH
        REPO intrepidcs/libicsneo
        HEAD_REF main
    )

    vcpkg_cmake_configure(
        SOURCE_PATH "${SOURCE_PATH}"
        OPTIONS
            -DLIBICSNEO_BUILD_EXAMPLES=OFF
            -DLIBICSNEO_BUILD_UNIT_TESTS=OFF
            -DLIBICSNEO_BUILD_DOCS=OFF
            -DLIBICSNEO_BUILD_ICSNEOC=OFF
            -DLIBICSNEO_BUILD_ICSNEOLEGACY=OFF
    )

    vcpkg_cmake_install()
    vcpkg_cmake_config_fixup(PACKAGE_NAME libicsneo CONFIG_PATH lib/cmake/libicsneo)
    vcpkg_copy_pdbs()

    file(REMOVE_RECURSE "${CURRENT_PACKAGES_DIR}/debug/include")
    file(INSTALL "${SOURCE_PATH}/LICENSE" DESTINATION "${CURRENT_PACKAGES_DIR}/share/${PORT}" RENAME copyright)

Using ``HEAD_REF main`` tracks the latest commit on the main branch. For reproducible
builds, replace it with a specific ``REF`` and ``SHA512``.

Project Configuration
=====================

Add ``libicsneo`` to your ``vcpkg.json`` dependencies:

.. code-block:: json

    {
      "name": "my-project",
      "dependencies": ["libicsneo"]
    }

Configure CMake with the overlay port path:

**Windows (x64-windows-static-md triplet required):**

.. code-block:: powershell

    cmake -B build -S . `
      -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake `
      -DVCPKG_TARGET_TRIPLET=x64-windows-static-md `
      -DVCPKG_OVERLAY_PORTS=./ports

**Linux:**

.. code-block:: bash

    cmake -B build -S . \
      -DCMAKE_TOOLCHAIN_FILE=$HOME/vcpkg/scripts/buildsystems/vcpkg.cmake \
      -DVCPKG_OVERLAY_PORTS=./ports

CMake Usage
===========

In your ``CMakeLists.txt``:

.. code-block:: cmake

    find_package(libicsneo CONFIG REQUIRED)
    target_link_libraries(my_app PRIVATE libicsneo::icsneocpp)

Linux udev Rules
================

To access USB devices without root, install the udev rules from the libicsneo
source (available after vcpkg build):

.. code-block:: bash

    sudo cp 99-intrepidcs.rules /etc/udev/rules.d/
    sudo udevadm control --reload-rules
    sudo udevadm trigger
