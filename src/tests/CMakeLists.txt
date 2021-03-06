set(ROOT_DIR ${CMAKE_CURRENT_LIST_DIR}/../..)

# Common dependencies
SET(PLATFORM_LIBS)

# Platform specific libraries
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    find_library(IOKIT_FRAMEWORK IOKit)
    find_library(COREFOUNDATION_FRAMEWORK CoreFoundation)
    #find_library(QUARTZCORE QuartzCore)
    find_library(APPKIT_FRAMEWORK AppKit)
    #find_library(QTKIT QTKit)
    find_library(AVFOUNDATION AVFoundation)
    find_library(IOBLUETOOTH IOBluetooth)
    #stdc++ ${QUARTZCORE} ${APPKIT_FRAMEWORK} ${QTKIT} ${AVFOUNDATION}
    list(APPEND PLATFORM_LIBS
        ${COREFOUNDATION_FRAMEWORK}
        ${IOKIT_FRAMEWORK}
        ${APPKIT_FRAMEWORK}
        ${AVFOUNDATION}
        ${IOBLUETOOTH})
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    #OpenCV extra dependencies: comctl32 gdi32 ole32 setupapi ws2_32 vfw32
    #setupapi required by hidapi
    #dinput8 required by libstem_gamepad
    list(APPEND PLATFORM_LIBS setupapi dinput8)
    IF(MINGW)
        #list(APPEND PLATFORM_LIBS stdc++)
    ENDIF(MINGW)
ELSE() #Linux
ENDIF()

IF(MSVC) 
# Disable asio auto linking in date-time and regex
add_definitions(-DBOOST_DATE_TIME_NO_LIB)
add_definitions(-DBOOST_REGEX_NO_LIB)
# fix: fatal error C1128: number of sections exceeded object file format limit
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj")
ENDIF()

#
# TEST_CAMERA and TEST_CAMERA_PARALLEL
#

SET(TEST_CAMERA_SRC)
SET(TEST_CAMERA_INCL_DIRS)
SET(TEST_CAMERA_REQ_LIBS)

# Boost
FIND_PACKAGE(Boost REQUIRED QUIET COMPONENTS atomic)
list(APPEND TEST_CAMERA_INCL_DIRS ${Boost_INCLUDE_DIRS})
list(APPEND TEST_CAMERA_REQ_LIBS ${Boost_LIBRARIES})

# OpenCV
set(OpenCV_STATIC ON)
IF(NOT(${CMAKE_SYSTEM_NAME} MATCHES "Windows"))
FIND_PACKAGE(OpenCV REQUIRED)
ENDIF()
list(APPEND TEST_CAMERA_INCL_DIRS ${OpenCV_INCLUDE_DIRS})
list(APPEND TEST_CAMERA_REQ_LIBS ${OpenCV_LIBS})

# PS3EYEDriver - only necessary on Mac and Win64, but can be used in Win32 (I think)
IF (${CMAKE_SYSTEM_NAME} MATCHES "Darwin"
    OR (${CMAKE_SYSTEM_NAME} MATCHES "Windows"))
    #PS3EYEDriver
    list(APPEND TEST_CAMERA_INCL_DIRS ${ROOT_DIR}/thirdparty/PS3EYEDriver/src)
    list(APPEND TEST_CAMERA_SRC
        ${ROOT_DIR}/thirdparty/PS3EYEDriver/src/ps3eye.h
        ${ROOT_DIR}/thirdparty/PS3EYEDriver/src/ps3eye.cpp)
    #Requires libusb
    find_package(USB1 REQUIRED)
    list(APPEND TEST_CAMERA_INCL_DIRS ${LIBUSB_INCLUDE_DIR})
    list(APPEND TEST_CAMERA_REQ_LIBS ${LIBUSB_LIBRARIES})
    add_definitions(-DHAVE_PS3EYE)
    IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")    
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-braced-scalar-init")
    ENDIF()
ENDIF()

# CL EYE - only on Win32
SET(ISWIN32 FALSE)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows"
    AND NOT(${CMAKE_C_SIZEOF_DATA_PTR} EQUAL 8))
    SET(ISWIN32 TRUE)
    add_definitions(-DHAVE_CLEYE)
    list(APPEND TEST_CAMERA_INCL_DIRS ${ROOT_DIR}/thirdparty/CLEYE)
    list(APPEND TEST_CAMERA_REQ_LIBS ${ROOT_DIR}/thirdparty/CLEYE/x86/lib/CLEyeMulticam.lib)
    find_path(CL_EYE_SDK_PATH CLEyeMulticam.dll
        HINTS C:/Windows/SysWOW64)
    #The non-Multicam version does not require any libs/dlls/includes
    #Uses OpenCV for video. Uses the registry for settings (maybe OpenCV for settings?)
    #But libusb is required for enumerating the devices and checking for the CL Eye Driver.
    find_package(USB1 REQUIRED)
    list(APPEND TEST_CAMERA_INCL_DIRS ${LIBUSB_INCLUDE_DIR})
    list(APPEND TEST_CAMERA_REQ_LIBS ${LIBUSB_LIBRARIES})

    # Windows utilities for querying driver infomation (provider name)
    list(APPEND TEST_CAMERA_INCL_DIRS ${ROOT_DIR}/src/psmoveservice/Device/Interface)
    list(APPEND TEST_CAMERA_INCL_DIRS ${ROOT_DIR}/src/psmoveservice/Server)
    list(APPEND TEST_CAMERA_INCL_DIRS ${ROOT_DIR}/src/psmoveservice/Platform)
    list(APPEND TEST_CAMERA_SRC ${ROOT_DIR}/src/psmoveservice/Device/Interface/DevicePlatformInterface.h)
    list(APPEND TEST_CAMERA_SRC ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.h)
    list(APPEND TEST_CAMERA_SRC ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.cpp)
    list(APPEND TEST_CAMERA_SRC ${ROOT_DIR}/src/psmoveservice/Platform/PlatformDeviceAPIWin32.h)
    list(APPEND TEST_CAMERA_SRC ${ROOT_DIR}/src/psmoveservice/Platform/PlatformDeviceAPIWin32.cpp)   
ENDIF()

# Our custom OpenCV VideoCapture classes
# We are not including the PSMoveService project on purpose.
list(APPEND TEST_CAMERA_INCL_DIRS 
    ${ROOT_DIR}/src/psmoveclient/
    ${ROOT_DIR}/src/psmoveservice/PSMoveTracker/PSEye)
list(APPEND TEST_CAMERA_SRC
    ${ROOT_DIR}/src/psmoveclient/ClientConstants.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveTracker/PSEye/PSEyeVideoCapture.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveTracker/PSEye/PSEyeVideoCapture.cpp)

# The test_camera app
add_executable(test_camera ${CMAKE_CURRENT_LIST_DIR}/test_camera.cpp ${TEST_CAMERA_SRC})
target_include_directories(test_camera PUBLIC ${TEST_CAMERA_INCL_DIRS})
target_link_libraries(test_camera ${PLATFORM_LIBS} ${TEST_CAMERA_REQ_LIBS})
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
add_dependencies(test_camera opencv)
ENDIF()
SET_TARGET_PROPERTIES(test_camera PROPERTIES FOLDER Test)    
    
# The test_camera_parallel app
add_executable(test_camera_parallel ${CMAKE_CURRENT_LIST_DIR}/test_camera_parallel.cpp ${TEST_CAMERA_SRC})
target_include_directories(test_camera_parallel PUBLIC ${TEST_CAMERA_INCL_DIRS})
target_link_libraries(test_camera_parallel ${PLATFORM_LIBS} ${TEST_CAMERA_REQ_LIBS})
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
add_dependencies(test_camera_parallel opencv)
ENDIF()
SET_TARGET_PROPERTIES(test_camera_parallel PROPERTIES FOLDER Test)

IF(${ISWIN32})
    IF(${CL_EYE_SDK_PATH} STREQUAL "CL_EYE_SDK_PATH-NOTFOUND")
        #If the developer does not have CLEyeMulticam.dll on their system,
        #copy it to the correct directory to prevent crashes.
        #If we distribute binaries (e.g., a server to use alongside a UE4 plugin)
        #then we will distribute it with this DLL with the server exe.
        #It will be up to CLEYE SDK users to delete this version of the DLL
        #to use their system version.
        add_custom_command(TARGET test_camera POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${ROOT_DIR}/thirdparty/CLEYE/x86/bin/CLEyeMulticam.dll"
                $<TARGET_FILE_DIR:test_camera>)                
        add_custom_command(TARGET test_camera_parallel POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${ROOT_DIR}/thirdparty/CLEYE/x86/bin/CLEyeMulticam.dll"
                $<TARGET_FILE_DIR:test_camera_parallel>)
    ENDIF()#CL_EYE not found
ENDIF()#ISWIN32

# Install    
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    install(TARGETS test_camera
        RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
        LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
        ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
    install(TARGETS test_camera_parallel
        RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
        LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
        ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# Test PSMove Controller
#

SET(TEST_PSMOVE_SRC)
SET(TEST_PSMOVE_INCL_DIRS)
SET(TEST_PSMOVE_REQ_LIBS)

# Dependencies

# hidapi
include_directories(${ROOT_DIR}/thirdparty/hidapi/hidapi)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/windows/hid.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/mac/hid.c)      
ELSE()
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/linux/hid.c)
ENDIF()
list(APPEND TEST_PSMOVE_SRC ${HIDAPI_SRC})

# libusb
find_package(USB1 REQUIRED)
list(APPEND TEST_PSMOVE_INCL_DIRS ${LIBUSB_INCLUDE_DIR})
list(APPEND TEST_PSMOVE_REQ_LIBS ${LIBUSB_LIBRARIES})

#Bluetooth
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_PSMOVE_SRC ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueriesOSX.mm)
ELSE()
ENDIF()

# libstem_gamepad
include_directories(${ROOT_DIR}/thirdparty/libstem_gamepad/source)
list(APPEND TEST_PSMOVE_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_private.c)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    list(APPEND TEST_PSMOVE_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_windows_dinput.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_PSMOVE_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_macosx.c)
ELSE()
    list(APPEND TEST_PSMOVE_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_linux.c)
ENDIF()

# Boost
# TODO: Eliminate boost::filesystem with C++14
FIND_PACKAGE(Boost REQUIRED QUIET COMPONENTS atomic chrono filesystem program_options system thread)
list(APPEND TEST_PSMOVE_INCL_DIRS ${Boost_INCLUDE_DIRS})
list(APPEND TEST_PSMOVE_REQ_LIBS ${Boost_LIBRARIES})

# Eigen math library
list(APPEND TEST_PSMOVE_INCL_DIRS ${EIGEN3_INCLUDE_DIR})

# PSMoveController
# We are not including the PSMoveService target on purpose, because this only tests
# a small part of the service and should not depend on the whole thing building.
list(APPEND TEST_PSMOVE_INCL_DIRS
    ${ROOT_DIR}/src/psmovemath/
    ${ROOT_DIR}/src/psmoveservice/
    ${ROOT_DIR}/src/psmoveservice/Server
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator
    ${ROOT_DIR}/src/psmoveservice/Device/Interface
    ${ROOT_DIR}/src/psmoveservice/Device/Manager
    ${ROOT_DIR}/src/psmoveservice/Device/USB    
    ${ROOT_DIR}/src/psmoveservice/Platform
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig
    ${ROOT_DIR}/src/psmoveservice/PSMoveController)
list(APPEND TEST_PSMOVE_SRC    
    ${ROOT_DIR}/src/psmovemath/MathAlignment.h
    ${ROOT_DIR}/src/psmovemath/MathAlignment.cpp
    ${ROOT_DIR}/src/psmovemath/MathEigen.h
    ${ROOT_DIR}/src/psmovemath/MathEigen.cpp    
    ${ROOT_DIR}/src/psmovemath/MathUtility.h
    ${ROOT_DIR}/src/psmovemath/MathUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.cpp    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.h
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/NullUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBBulkTransferBundle.cpp        
    ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueries.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.cpp
    ${ROOT_DIR}/src/psmoveservice/PSMoveController/PSMoveController.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveController/PSMoveController.cpp)

# psmoveprotocol
list(APPEND TEST_PSMOVE_INCL_DIRS ${ROOT_DIR}/src/psmoveprotocol)
list(APPEND TEST_PSMOVE_REQ_LIBS PSMoveProtocol)

add_executable(test_psmove_controller ${CMAKE_CURRENT_LIST_DIR}/test_psmove_controller.cpp ${TEST_PSMOVE_SRC})
target_include_directories(test_psmove_controller PUBLIC ${TEST_PSMOVE_INCL_DIRS})
target_link_libraries(test_psmove_controller ${PLATFORM_LIBS} ${TEST_PSMOVE_REQ_LIBS})
SET_TARGET_PROPERTIES(test_psmove_controller PROPERTIES FOLDER Test)

# Install    
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    install(TARGETS test_psmove_controller
        RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
        LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
        ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# Test Navi Controller
#

SET(TEST_NAVI_SRC)
SET(TEST_NAVI_INCL_DIRS)
SET(TEST_NAVI_REQ_LIBS)

# Dependencies

# hidapi
include_directories(${ROOT_DIR}/thirdparty/hidapi/hidapi)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/windows/hid.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/mac/hid.c)      
ELSE()
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/linux/hid.c)
ENDIF()
list(APPEND TEST_NAVI_SRC ${HIDAPI_SRC})

# libusb
find_package(USB1 REQUIRED)
list(APPEND TEST_NAVI_INCL_DIRS ${LIBUSB_INCLUDE_DIR})
list(APPEND TEST_NAVI_REQ_LIBS ${LIBUSB_LIBRARIES})

#Bluetooth
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_NAVI_SRC ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueriesOSX.mm)
ELSE()
ENDIF()

# libstem_gamepad
include_directories(${ROOT_DIR}/thirdparty/libstem_gamepad/source)
list(APPEND TEST_NAVI_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_private.c)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    list(APPEND TEST_NAVI_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_windows_dinput.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_NAVI_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_macosx.c)
ELSE()
    list(APPEND TEST_NAVI_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_linux.c)
ENDIF()

# Boost
# TODO: Eliminate boost::filesystem with C++14
FIND_PACKAGE(Boost REQUIRED QUIET COMPONENTS atomic chrono filesystem program_options system thread)
list(APPEND TEST_NAVI_INCL_DIRS ${Boost_INCLUDE_DIRS})
list(APPEND TEST_NAVI_REQ_LIBS ${Boost_LIBRARIES})

# Eigen math library
list(APPEND TEST_NAVI_INCL_DIRS ${EIGEN3_INCLUDE_DIR})

# PSMoveController
# We are not including the PSMoveService target on purpose, because this only tests
# a small part of the service and should not depend on the whole thing building.
list(APPEND TEST_NAVI_INCL_DIRS
    ${ROOT_DIR}/src/psmovemath/
    ${ROOT_DIR}/src/psmoveservice/
    ${ROOT_DIR}/src/psmoveservice/Server
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator
    ${ROOT_DIR}/src/psmoveservice/Device/Interface
    ${ROOT_DIR}/src/psmoveservice/Device/Manager
    ${ROOT_DIR}/src/psmoveservice/Device/USB
    ${ROOT_DIR}/src/psmoveservice/Platform
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig
    ${ROOT_DIR}/src/psmoveservice/PSNaviController)
list(APPEND TEST_NAVI_SRC    
    ${ROOT_DIR}/src/psmovemath/MathAlignment.h
    ${ROOT_DIR}/src/psmovemath/MathAlignment.cpp
    ${ROOT_DIR}/src/psmovemath/MathEigen.h
    ${ROOT_DIR}/src/psmovemath/MathEigen.cpp    
    ${ROOT_DIR}/src/psmovemath/MathUtility.h
    ${ROOT_DIR}/src/psmovemath/MathUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.cpp        
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.h
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/NullUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBBulkTransferBundle.cpp
    ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueries.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.cpp
    ${ROOT_DIR}/src/psmoveservice/PSNaviController/PSNaviController.h
    ${ROOT_DIR}/src/psmoveservice/PSNaviController/PSNaviController.cpp)

# psmoveprotocol
list(APPEND TEST_NAVI_INCL_DIRS ${ROOT_DIR}/src/psmoveprotocol)
list(APPEND TEST_NAVI_REQ_LIBS PSMoveProtocol)

add_executable(test_navi_controller ${CMAKE_CURRENT_LIST_DIR}/test_navi_controller.cpp ${TEST_NAVI_SRC})
target_include_directories(test_navi_controller PUBLIC ${TEST_NAVI_INCL_DIRS})
target_link_libraries(test_navi_controller ${PLATFORM_LIBS} ${TEST_NAVI_REQ_LIBS})
SET_TARGET_PROPERTIES(test_navi_controller PROPERTIES FOLDER Test)

# Install    
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    install(TARGETS test_navi_controller
        RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
        LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
        ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# Test DS4 Controller
#

SET(TEST_DS4_CTRLR_SRC)
SET(TEST_DS4_CTRLR_INCL_DIRS)
SET(TEST_DS4_CTRLR_REQ_LIBS)

# Dependencies

# Platform specific libraries
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    #hid required for HidD_SetOutputReport() in DualShock4 controller
    list(APPEND TEST_DS4_CTRLR_REQ_LIBS bthprops hid)
ELSE() #Linux
ENDIF()

# hidapi
include_directories(${ROOT_DIR}/thirdparty/hidapi/hidapi)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/windows/hid.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/mac/hid.c)      
ELSE()
    set(HIDAPI_SRC ${ROOT_DIR}/thirdparty/hidapi/linux/hid.c)
ENDIF()
list(APPEND TEST_DS4_CTRLR_SRC ${HIDAPI_SRC})

# libusb
find_package(USB1 REQUIRED)
list(APPEND TEST_DS4_CTRLR_INCL_DIRS ${LIBUSB_INCLUDE_DIR})
list(APPEND TEST_DS4_CTRLR_REQ_LIBS ${LIBUSB_LIBRARIES})

#Bluetooth
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueriesWin32.cpp)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueriesOSX.mm)
ELSE()
ENDIF()

# libstem_gamepad
include_directories(${ROOT_DIR}/thirdparty/libstem_gamepad/source)
list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_private.c)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_windows_dinput.c)
ELSEIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_macosx.c)
ELSE()
    list(APPEND TEST_DS4_CTRLR_SRC ${ROOT_DIR}/thirdparty/libstem_gamepad/source/gamepad/Gamepad_linux.c)
ENDIF()

# Boost
# TODO: Eliminate boost::filesystem with C++14
FIND_PACKAGE(Boost REQUIRED QUIET COMPONENTS atomic chrono filesystem program_options system thread)
list(APPEND TEST_DS4_CTRLR_INCL_DIRS ${Boost_INCLUDE_DIRS})
list(APPEND TEST_DS4_CTRLR_REQ_LIBS ${Boost_LIBRARIES})

# Eigen math library
list(APPEND TEST_DS4_CTRLR_INCL_DIRS ${EIGEN3_INCLUDE_DIR})

# PSMoveController
# We are not including the PSMoveService target on purpose, because this only tests
# a small part of the service and should not depend on the whole thing building.
list(APPEND TEST_DS4_CTRLR_INCL_DIRS
    ${ROOT_DIR}/src/psmovemath/
    ${ROOT_DIR}/src/psmoveservice/
    ${ROOT_DIR}/src/psmoveservice/Server
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator
    ${ROOT_DIR}/src/psmoveservice/Device/Interface
    ${ROOT_DIR}/src/psmoveservice/Device/Manager
    ${ROOT_DIR}/src/psmoveservice/Device/USB
    ${ROOT_DIR}/src/psmoveservice/Platform
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig
    ${ROOT_DIR}/src/psmoveservice/PSDualShock4)
list(APPEND TEST_DS4_CTRLR_SRC    
    ${ROOT_DIR}/src/psmovemath/MathAlignment.h
    ${ROOT_DIR}/src/psmovemath/MathAlignment.cpp
    ${ROOT_DIR}/src/psmovemath/MathEigen.h
    ${ROOT_DIR}/src/psmovemath/MathEigen.cpp    
    ${ROOT_DIR}/src/psmovemath/MathUtility.h
    ${ROOT_DIR}/src/psmovemath/MathUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerGamepadEnumerator.cpp        
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerHidDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.h    
    ${ROOT_DIR}/src/psmoveservice/Device/Enumerator/ControllerUSBDeviceEnumerator.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.h
    ${ROOT_DIR}/src/psmoveservice/Device/Manager/USBDeviceManager.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/NullUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBApi.cpp
    ${ROOT_DIR}/src/psmoveservice/Device/USB/LibUSBBulkTransferBundle.cpp
    ${ROOT_DIR}/src/psmoveservice/Platform/BluetoothQueries.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.h
    ${ROOT_DIR}/src/psmoveservice/PSMoveConfig/PSMoveConfig.cpp
    ${ROOT_DIR}/src/psmoveservice/PSDualShock4/PSDualShock4Controller.h
    ${ROOT_DIR}/src/psmoveservice/PSDualShock4/PSDualShock4Controller.cpp)

# psmoveprotocol
list(APPEND TEST_DS4_CTRLR_INCL_DIRS ${ROOT_DIR}/src/psmoveprotocol)
list(APPEND TEST_DS4_CTRLR_REQ_LIBS PSMoveProtocol)

add_executable(test_ds4_controller ${CMAKE_CURRENT_LIST_DIR}/test_ds4_controller.cpp ${TEST_DS4_CTRLR_SRC})
target_include_directories(test_ds4_controller PUBLIC ${TEST_DS4_CTRLR_INCL_DIRS})
target_link_libraries(test_ds4_controller ${PLATFORM_LIBS} ${TEST_DS4_CTRLR_REQ_LIBS})
SET_TARGET_PROPERTIES(test_ds4_controller PROPERTIES FOLDER Test)

# Install    
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    install(TARGETS test_ds4_controller
        RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
        LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
        ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# TEST_CONSOLE_CAPI
#
add_executable(test_console_CAPI test_console_CAPI.cpp)
target_include_directories(test_console_CAPI PUBLIC ${ROOT_DIR}/src/psmoveclient/)
target_link_libraries(test_console_CAPI PSMoveClient_CAPI)
SET_TARGET_PROPERTIES(test_console_CAPI PROPERTIES FOLDER Test)
# Install
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
install(TARGETS test_console_CAPI
    RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
    LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
    ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# TEST_KALMAN_FILTER
#

list(APPEND TEST_KALMAN_INCL_DIRS
    ${ROOT_DIR}/src/psmovemath/
    ${ROOT_DIR}/src/psmoveservice/Device/Interface
    ${ROOT_DIR}/src/psmoveservice/Filter/
    ${ROOT_DIR}/src/psmoveservice/PSMoveController
    ${ROOT_DIR}/src/psmoveservice/Server/)
list(APPEND TEST_KALMAN_SRC    
    ${ROOT_DIR}/src/psmovemath/MathAlignment.h
    ${ROOT_DIR}/src/psmovemath/MathAlignment.cpp
    ${ROOT_DIR}/src/psmovemath/MathEigen.h
    ${ROOT_DIR}/src/psmovemath/MathEigen.cpp    
    ${ROOT_DIR}/src/psmovemath/MathUtility.h
    ${ROOT_DIR}/src/psmovemath/MathUtility.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/CompoundPoseFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/CompoundPoseFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanOrientationFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanOrientationFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanPositionFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanPositionFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanPoseFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/KalmanPoseFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/OrientationFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/OrientationFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/PoseFilterInterface.h
    ${ROOT_DIR}/src/psmoveservice/Filter/PoseFilterInterface.cpp
    ${ROOT_DIR}/src/psmoveservice/Filter/PositionFilter.h
    ${ROOT_DIR}/src/psmoveservice/Filter/PositionFilter.cpp
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.h
    ${ROOT_DIR}/src/psmoveservice/Server/ServerLog.cpp)
    
# Eigen math library
list(APPEND TEST_KALMAN_INCL_DIRS ${EIGEN3_INCLUDE_DIR})   
list(APPEND TEST_KALMAN_INCL_DIRS ${ROOT_DIR}/thirdparty/kalman/include/)

add_executable(test_kalman_filter ${CMAKE_CURRENT_LIST_DIR}/test_kalman_filter.cpp ${TEST_KALMAN_SRC})
target_include_directories(test_kalman_filter PUBLIC ${TEST_KALMAN_INCL_DIRS})
SET_TARGET_PROPERTIES(test_kalman_filter PROPERTIES FOLDER Test)

# Install
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
install(TARGETS test_kalman_filter
RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()

#
# UNIT_TESTS
#

list(APPEND UNIT_TEST_INCL_DIRS
    ${ROOT_DIR}/src/psmovemath/)

# Eigen math library
list(APPEND UNIT_TEST_INCL_DIRS ${EIGEN3_INCLUDE_DIR})   

list(APPEND UNIT_TEST_SRC    
    ${ROOT_DIR}/src/psmovemath/MathAlignment.h
    ${ROOT_DIR}/src/psmovemath/MathAlignment.cpp
    ${ROOT_DIR}/src/psmovemath/MathEigen.h
    ${ROOT_DIR}/src/psmovemath/MathEigen.cpp	
    ${ROOT_DIR}/src/psmovemath/MathUtility.h
    ${ROOT_DIR}/src/psmovemath/MathUtility.cpp
    ${ROOT_DIR}/src/tests/math_alignment_unit_tests.cpp
    ${ROOT_DIR}/src/tests/math_eigen_unit_tests.cpp
    ${ROOT_DIR}/src/tests/math_utility_unit_tests.cpp
    ${ROOT_DIR}/src/tests/unit_test.h)
    
add_executable(unit_test_suite ${CMAKE_CURRENT_LIST_DIR}/unit_test_suite.cpp ${UNIT_TEST_SRC})
target_include_directories(unit_test_suite PUBLIC ${UNIT_TEST_INCL_DIRS})
SET_TARGET_PROPERTIES(unit_test_suite PROPERTIES FOLDER Test)

# Install
IF(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
install(TARGETS unit_test_suite
RUNTIME DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/bin
LIBRARY DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib
ARCHIVE DESTINATION ${ROOT_DIR}/${PSM_PROJECT_NAME}/${ARCH_LABEL}/lib)
ELSE() #Linux/Darwin
ENDIF()


#
# Test hidapi in MacOS Sierra
#
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
add_executable(test_hidapi_sierra
    ${CMAKE_CURRENT_LIST_DIR}/test_hidapi_sierra.cpp
    ${ROOT_DIR}/thirdparty/hidapi/mac/hid.c)
target_include_directories(test_hidapi_sierra
    PUBLIC
    ${ROOT_DIR}/thirdparty/hidapi/hidapi)
    #/usr/local/opt/hidapi/include/hidapi
target_link_libraries(test_hidapi_sierra ${PLATFORM_LIBS})
#target_link_libraries(test_hidapi_sierra /usr/local/opt/hidapi/lib/libhidapi.dylib)
SET_TARGET_PROPERTIES(test_hidapi_sierra PROPERTIES FOLDER Test)
ENDIF()