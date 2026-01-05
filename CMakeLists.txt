cmake_minimum_required(VERSION 3.5)

project(smartcam)

# --- Enable GLIB-based VVAS utils for all modules ---
add_compile_definitions(VVAS_GLIB_UTILS=1)
add_compile_definitions(XLNX_EMBEDDED_PLATFORM=1)   # Also expected by VVAS
# ----------------------------------------------------
SET(CMAKE_INSTALL_PREFIX $ENV{OECORE_TARGET_SYSROOT})

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-narrowing -fpermissive -rdynamic -Wl,--no-undefined")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread -std=c++14")

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
find_package(GStreamer REQUIRED)
find_package(OpenCV REQUIRED)

SET(INSTALL_PATH "opt/xilinx/kv260-smartcam")

# If you keep using versioned .so names, define the paths once
set(VVAS_UTIL_LIB        "${CMAKE_SYSROOT}/usr/lib/libvvasutil-3.0.so")
set(VVAS_INFERMETA_LIB   "${CMAKE_SYSROOT}/usr/lib/libgstvvasinfermeta-3.0.so")
set(VVAS_STRUCTURE_LIB   "${CMAKE_SYSROOT}/usr/lib/libvvasstructure-3.0.so")
set(VVAS_COREUTILS_LIB   "${CMAKE_SYSROOT}/usr/lib/libvvascore_utils-1.0.so")

# vvas_xpp library
add_library(vvas_xpp SHARED src/vvas_xpp_pipeline.c)
target_include_directories(vvas_xpp PRIVATE ${GSTREAMER_INCLUDE_DIRS})
target_link_libraries(vvas_xpp
    jansson
    ${VVAS_UTIL_LIB}
    ${VVAS_INFERMETA_LIB}
)
install(TARGETS vvas_xpp DESTINATION ${INSTALL_PATH}/lib)

# vvas_airender library
add_library(vvas_airender SHARED src/vvas_airender.cpp)
target_include_directories(vvas_airender PRIVATE ${GSTREAMER_INCLUDE_DIRS})
target_link_libraries(vvas_airender
    jansson
    ${VVAS_UTIL_LIB}
    ${VVAS_INFERMETA_LIB}
    ${VVAS_STRUCTURE_LIB}
    ${VVAS_COREUTILS_LIB}
    gstreamer-1.0
    gstbase-1.0
    glib-2.0
    opencv_core
    opencv_video
    opencv_videoio
    opencv_imgproc
)
install(TARGETS vvas_airender DESTINATION ${INSTALL_PATH}/lib)

# smartcam executable
add_executable(${CMAKE_PROJECT_NAME} src/main.cpp)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${GSTREAMER_INCLUDE_DIRS})
target_link_libraries(${CMAKE_PROJECT_NAME}
    gstapp-1.0 gstreamer-1.0 gstbase-1.0 gobject-2.0 glib-2.0 gstvideo-1.0 gstallocators-1.0 gstrtsp-1.0 gstrtspserver-1.0
    glib-2.0 gobject-2.0 )		

install(TARGETS ${CMAKE_PROJECT_NAME} DESTINATION ${INSTALL_PATH}/bin)

install(PROGRAMS
    script/01.mipi-rtsp.sh
    script/02.mipi-dp.sh
    script/03.file-file.sh
    script/04.file-ssd-dp.sh
    script/smartcam-install.py
    DESTINATION ${INSTALL_PATH}/bin)

install(FILES
    README
    DESTINATION ${INSTALL_PATH}/
    RENAME      README_SMARTCAM
    )

install(DIRECTORY
    config/facedetect
    config/refinedet
    config/ssd
    DESTINATION ${INSTALL_PATH}/share/vvas/)

install(DIRECTORY
    models/kv260/densebox_640_360
	models/kv260/refinedet_pruned_0_96
	models/kv260/ssd_adas_pruned_0_95
    DESTINATION ${INSTALL_PATH}/share/vitis_ai_library/models/)

install(DIRECTORY
    notebook/
    DESTINATION ${INSTALL_PATH}/share/notebooks/)

set(VERSION "1.0.1")
set(CPACK_PACKAGE_VERSION ${VERSION})
set(CPACK_GENERATOR "RPM")
set(CPACK_PACKAGE_NAME "${CMAKE_PROJECT_NAME}")
set(CPACK_PACKAGE_RELEASE 1)
set(CPACK_PACKAGE_CONTACT "Yuxiz@xilinx.com")
set(CPACK_PACKAGE_VENDOR "xilinx")
set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "arm64")
set(CPACK_RPM_PACKAGE_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})
set(CPACK_PACKAGING_INSTALL_PREFIX "/")
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_PACKAGE_RELEASE}.${CMAKE_SYSTEM_PROCESSOR}")
set(CPACK_RPM_SPEC_MORE_DEFINE "%define _build_id_links none")
include(CPack)