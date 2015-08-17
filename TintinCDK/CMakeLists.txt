CMAKE_MINIMUM_REQUIRED(VERSION 2.8.11)
PROJECT(TintinCDKCamera)
set(TINTINCDK_MAJOR_VERSION 0)
set(TINTINCDK_MINOR_VERSION 5)
set(TINTINCDK_PATCH_VERSION 2)
set(TINTINCDK_VERSION ${TINTINCDK_MAJOR_VERSION}.${TINTINCDK_MINOR_VERSION}.${TINTINCDK_PATCH_VERSION})

FIND_PACKAGE(TI3DToF 0.5.2 REQUIRED)

### Do not export any symbol by default
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN 1)
include(GenerateExportHeader)

set(CMAKE_INCLUDE_CURRENT_DIR 1)

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  SET(LINUX 1)
  add_definitions(-msse2 -pthread -std=c++11 -fPIC -ffast-math)
  ADD_DEFINITIONS(-DLINUX)  

  if(ARCH STREQUAL "x86_64")
    set(ARCH amd64)
  endif()
  
  file(WRITE "${CMAKE_BINARY_DIR}/make_deb.sh" "for i in CPackConfig-*.cmake; do cpack -G DEB --config $i; done\n")
  
  SET(COMMON_LIB_FILES)
  
ELSEIF(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  SET(WINDOWS 1)
  add_definitions(/fp:fast /W0 /bigobj)
  ADD_DEFINITIONS(-DWINDOWS /wd4251)
  SET(COMMON_LIBS Version)

  configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/dllversion.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/dllversion.rc
    @ONLY)
  
  SET(COMMON_LIB_FILES ${CMAKE_CURRENT_BINARY_DIR}/dllversion.rc)
  
endif()

add_library(tintincdk SHARED
  TintinCDKCamera.cpp
  TintinCDKCameraFactory.cpp
  TintinCDKDownloaderFactory.cpp
  TintinEEPROMDownloader.cpp
  ${COMMON_LIB_FILES}
)

generate_export_header(tintincdk
  EXPORT_FILE_NAME "SymbolExports.h"
  EXPORT_MACRO_NAME SYMBOL_EXPORT
)

target_include_directories(tintincdk PUBLIC ${VOXEL_INCLUDE_DIRS} ${TI3DToF_INCLUDE_DIRS})

TARGET_LINK_LIBRARIES(tintincdk ${VOXEL_LIBRARIES} ${TI3DToF_LIBRARIES})

install(TARGETS tintincdk
  LIBRARY DESTINATION lib/voxel COMPONENT tintincdk
  ARCHIVE DESTINATION lib/voxel COMPONENT tintincdk
  RUNTIME DESTINATION lib/voxel COMPONENT tintincdk
)

list(APPEND CONF_FILES
  TintinCDKCamera.conf
  TintinCDKCameraLensOnly.conf
  TintinCDKCameraShortRange.conf
  TintinCDKCameraLongRange.conf
  TintinCDKCameraHighAmbient.conf
  TintinCDKCameraNoCalibration.conf
  TintinCDKCameraPhaseOffset.bin
)

list(APPEND UDEV_FILES
  72-TintinCDK.rules
)

set_target_properties(tintincdk PROPERTIES 
                      VERSION ${TINTINCDK_VERSION} 
                      SOVERSION ${VOXEL_ABI_VERSION})
                      
### Copy configuration and firmware files to build directory
foreach(f ${CONF_FILES} ${FW_FILES})
  get_filename_component(filename ${f} NAME)
  list(APPEND filesToCopy
    ${CMAKE_CURRENT_BINARY_DIR}/${filename}
  )
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${filename}
                   COMMAND ${CMAKE_COMMAND} -E copy
                   ${CMAKE_CURRENT_SOURCE_DIR}/${f} ${CMAKE_CURRENT_BINARY_DIR}/
                   MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/${f})
endforeach()

if(WINDOWS)
list(APPEND filesToCopy
    ${CMAKE_CURRENT_BINARY_DIR}/WinDrivers
  )
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/WinDrivers
                   COMMAND ${CMAKE_COMMAND} -E copy_directory
                   ${CMAKE_CURRENT_SOURCE_DIR}/WinDrivers ${CMAKE_CURRENT_BINARY_DIR}/WinDrivers
                   MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/WinDrivers)
                   
install(DIRECTORY
  ${CMAKE_CURRENT_BINARY_DIR}/WinDrivers
  DESTINATION Prerequisites
  COMPONENT tintincdk
)
endif()

add_custom_target(filesToCopyTarget ALL DEPENDS ${filesToCopy})
                   
IF(LINUX)
install(FILES 
  ${CONF_FILES}
  DESTINATION /etc/voxel/
  COMPONENT tintincdk
)

install(FILES 
  ${FW_FILES}
  DESTINATION /lib/firmware/voxel
  COMPONENT tintincdk
)

install(FILES
  ${UDEV_FILES}
  DESTINATION /etc/udev/rules.d/
  COMPONENT tintincdk
)
ELSEIF(WINDOWS)
INSTALL(FILES
  ${CONF_FILES}
  DESTINATION lib/voxel/conf
  COMPONENT tintincdk)

INSTALL(FILES
  ${FW_FILES}
  DESTINATION lib/voxel/fw
  COMPONENT tintincdk)
ENDIF()


