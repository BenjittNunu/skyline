cmake_minimum_required(VERSION 3.16)
project(Skyline LANGUAGES C CXX ASM VERSION 0.3)

set(BUILD_TESTS OFF CACHE BOOL "Build Tests" FORCE)
set(BUILD_TESTING OFF CACHE BOOL "Build Testing" FORCE)
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build Shared Libraries" FORCE)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

set(source_DIR ${CMAKE_SOURCE_DIR}/src/main/cpp)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing -Wno-unused-command-line-argument")
set(CMAKE_CXX_FLAGS_RELEASE "-Ofast -flto=full -fno-stack-protector -DNDEBUG")

# Build all libraries with -Ofast but with default debug data (-g) for debug builds
set(CMAKE_CXX_FLAGS_DEBUG "-Ofast")

# libcxx
set(ANDROID_STL "none")
set(LIBCXX_INCLUDE_TESTS OFF)
set(LIBCXX_INCLUDE_BENCHMARKS OFF)
set(LIBCXX_INCLUDE_DOCS OFF)
set(LIBCXX_ENABLE_SHARED FALSE)
set(LIBCXX_ENABLE_ASSERTIONS FALSE)
set(LIBCXX_STANDALONE_BUILD FALSE)
add_subdirectory("libraries/llvm/libcxx")
link_libraries(cxx_static)
get_target_property(LIBCXX_INCLUDE_COMPILE_OPTION cxx-headers INTERFACE_COMPILE_OPTIONS)
string(REGEX REPLACE "-I" "" LIBCXX_INCLUDE_DIRECTORY_LIST "${LIBCXX_INCLUDE_COMPILE_OPTION}")
list(GET LIBCXX_INCLUDE_DIRECTORY_LIST 1 LIBCXX_TARGET_INCLUDE_DIRECTORY) # We just want the target include directory
set_target_properties(cxx-headers PROPERTIES INTERFACE_COMPILE_OPTIONS -isystem${LIBCXX_TARGET_INCLUDE_DIRECTORY})

# libcxxabi
set(LIBCXXABI_INCLUDE_TESTS OFF)
set(LIBCXXABI_ENABLE_SHARED FALSE)
set(LIBCXXABI_STANDALONE_BUILD FALSE)
set(LIBCXXABI_LIBCXX_INCLUDES "${LIBCXX_TARGET_INCLUDE_DIRECTORY}" CACHE STRING "" FORCE)
add_subdirectory("libraries/llvm/libcxxabi")
link_libraries(cxxabi_static)

# {fmt}
add_subdirectory("libraries/fmt")

# TzCode
add_subdirectory("libraries/tzcode")
target_compile_options(tzcode PRIVATE -Wno-everything)

# Oboe
add_subdirectory("libraries/oboe")
include_directories(SYSTEM "libraries/oboe/include")

# LZ4
set(LZ4_BUILD_CLI OFF CACHE BOOL "Build LZ4 CLI" FORCE)
set(LZ4_BUILD_LEGACY_LZ4C OFF CACHE BOOL "Build lz4c progam with legacy argument support" FORCE)
add_subdirectory("libraries/lz4/build/cmake")
include_directories(SYSTEM "libraries/lz4/lib")

# Vulkan + Vulkan-Hpp
add_compile_definitions(VK_USE_PLATFORM_ANDROID_KHR) # We want all the Android-specific structures to be defined
add_compile_definitions(VULKAN_HPP_NO_SPACESHIP_OPERATOR) # libcxx doesn't implement operator<=> for std::array which breaks this
add_compile_definitions(VULKAN_HPP_NO_STRUCT_CONSTRUCTORS) # We want to use designated initializers in Vulkan-Hpp
add_compile_definitions(VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1) # We use the dynamic loader rather than the static one to avoid an additional level of indirection
include_directories(SYSTEM "libraries/vkhpp")
include_directories(SYSTEM "libraries/vkhpp/Vulkan-Headers/include") # We use base Vulkan headers from this to ensure version parity with Vulkan-Hpp

# Vulkan Memory Allocator
include_directories("libraries/vkma/include")
add_library(vkma STATIC libraries/vkma.cpp)
target_compile_options(vkma PRIVATE -Wno-everything)

# PugiXML (Header-only mode)
include_directories(SYSTEM "libraries/pugixml/src")

# Frozen
include_directories(SYSTEM "libraries/frozen/include")

# MbedTLS
set(ENABLE_TESTING OFF CACHE BOOL "Build mbed TLS tests." FORCE)
set(ENABLE_PROGRAMS OFF CACHE BOOL "Build mbed TLS programs." FORCE)
set(UNSAFE_BUILD ON CACHE BOOL "Allow unsafe builds. These builds ARE NOT SECURE." FORCE)
add_subdirectory("libraries/mbedtls")
include_directories(SYSTEM "libraries/mbedtls/include")
target_compile_options(mbedcrypto PRIVATE -Wno-everything)

# Opus
set(OPUS_INSTALL_CMAKE_CONFIG_MODULE OFF CACHE BOOL "Install Opus CMake package config module" FORCE)
include_directories(SYSTEM "libraries/opus/include")
add_subdirectory("libraries/opus")
target_compile_definitions(opus PRIVATE OPUS_WILL_BE_SLOW=1) # libopus will warn when built without optimizations

# Perfetto SDK
include_directories(SYSTEM "libraries/perfetto/sdk")
add_library(perfetto STATIC libraries/perfetto/sdk/perfetto.cc)
target_compile_options(perfetto PRIVATE -Wno-everything)

# Boost
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_MULTITHREADED ON)
add_subdirectory("libraries/boost")

# Build Skyline with full debugging data and -Og for debug builds
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g3 -glldb -gdwarf-5")

# Skyline
add_library(skyline SHARED
        ${source_DIR}/emu_jni.cpp
        ${source_DIR}/loader_jni.cpp
        ${source_DIR}/skyline/common.cpp
        ${source_DIR}/skyline/common/logger.cpp
        ${source_DIR}/skyline/common/settings.cpp
        ${source_DIR}/skyline/common/signal.cpp
        ${source_DIR}/skyline/common/uuid.cpp
        ${source_DIR}/skyline/common/trace.cpp
        ${source_DIR}/skyline/nce/guest.S
        ${source_DIR}/skyline/nce.cpp
        ${source_DIR}/skyline/jvm.cpp
        ${source_DIR}/skyline/os.cpp
        ${source_DIR}/skyline/kernel/memory.cpp
        ${source_DIR}/skyline/kernel/scheduler.cpp
        ${source_DIR}/skyline/kernel/ipc.cpp
        ${source_DIR}/skyline/kernel/svc.cpp
        ${source_DIR}/skyline/kernel/types/KProcess.cpp
        ${source_DIR}/skyline/kernel/types/KThread.cpp
        ${source_DIR}/skyline/kernel/types/KSharedMemory.cpp
        ${source_DIR}/skyline/kernel/types/KPrivateMemory.cpp
        ${source_DIR}/skyline/kernel/types/KSyncObject.cpp
        ${source_DIR}/skyline/audio.cpp
        ${source_DIR}/skyline/audio/track.cpp
        ${source_DIR}/skyline/audio/resampler.cpp
        ${source_DIR}/skyline/audio/adpcm_decoder.cpp
        ${source_DIR}/skyline/gpu.cpp
        ${source_DIR}/skyline/gpu/memory_manager.cpp
        ${source_DIR}/skyline/gpu/texture_manager.cpp
        ${source_DIR}/skyline/gpu/command_scheduler.cpp
        ${source_DIR}/skyline/gpu/texture/texture.cpp
        ${source_DIR}/skyline/gpu/presentation_engine.cpp
        ${source_DIR}/skyline/gpu/interconnect/command_executor.cpp
        ${source_DIR}/skyline/gpu/interconnect/command_nodes.cpp
        ${source_DIR}/skyline/soc/smmu.cpp
        ${source_DIR}/skyline/soc/host1x/syncpoint.cpp
        ${source_DIR}/skyline/soc/host1x/command_fifo.cpp
        ${source_DIR}/skyline/soc/host1x/classes/host1x.cpp
        ${source_DIR}/skyline/soc/host1x/classes/vic.cpp
        ${source_DIR}/skyline/soc/host1x/classes/nvdec.cpp
        ${source_DIR}/skyline/soc/gm20b/channel.cpp
        ${source_DIR}/skyline/soc/gm20b/gpfifo.cpp
        ${source_DIR}/skyline/soc/gm20b/gmmu.cpp
        ${source_DIR}/skyline/soc/gm20b/engines/gpfifo.cpp
        ${source_DIR}/skyline/soc/gm20b/engines/maxwell_3d.cpp
        ${source_DIR}/skyline/soc/gm20b/engines/maxwell/macro_interpreter.cpp
        ${source_DIR}/skyline/input/npad.cpp
        ${source_DIR}/skyline/input/npad_device.cpp
        ${source_DIR}/skyline/input/touch.cpp
        ${source_DIR}/skyline/crypto/aes_cipher.cpp
        ${source_DIR}/skyline/crypto/key_store.cpp
        ${source_DIR}/skyline/loader/loader.cpp
        ${source_DIR}/skyline/loader/nro.cpp
        ${source_DIR}/skyline/loader/nso.cpp
        ${source_DIR}/skyline/loader/nca.cpp
        ${source_DIR}/skyline/loader/xci.cpp
        ${source_DIR}/skyline/loader/nsp.cpp
        ${source_DIR}/skyline/vfs/partition_filesystem.cpp
        ${source_DIR}/skyline/vfs/ctr_encrypted_backing.cpp
        ${source_DIR}/skyline/vfs/rom_filesystem.cpp
        ${source_DIR}/skyline/vfs/os_filesystem.cpp
        ${source_DIR}/skyline/vfs/os_backing.cpp
        ${source_DIR}/skyline/vfs/android_asset_filesystem.cpp
        ${source_DIR}/skyline/vfs/android_asset_backing.cpp
        ${source_DIR}/skyline/vfs/nacp.cpp
        ${source_DIR}/skyline/vfs/npdm.cpp
        ${source_DIR}/skyline/vfs/nca.cpp
        ${source_DIR}/skyline/vfs/ticket.cpp
        ${source_DIR}/skyline/services/serviceman.cpp
        ${source_DIR}/skyline/services/base_service.cpp
        ${source_DIR}/skyline/services/sm/IUserInterface.cpp
        ${source_DIR}/skyline/services/fatalsrv/IService.cpp
        ${source_DIR}/skyline/services/audio/IAudioOutManager.cpp
        ${source_DIR}/skyline/services/audio/IAudioOut.cpp
        ${source_DIR}/skyline/services/audio/IAudioDevice.cpp
        ${source_DIR}/skyline/services/audio/IAudioRendererManager.cpp
        ${source_DIR}/skyline/services/audio/IAudioRenderer/IAudioRenderer.cpp
        ${source_DIR}/skyline/services/audio/IAudioRenderer/voice.cpp
        ${source_DIR}/skyline/services/audio/IAudioRenderer/memory_pool.cpp
        ${source_DIR}/skyline/services/settings/ISettingsServer.cpp
        ${source_DIR}/skyline/services/settings/ISystemSettingsServer.cpp
        ${source_DIR}/skyline/services/apm/IManager.cpp
        ${source_DIR}/skyline/services/apm/ISession.cpp
        ${source_DIR}/skyline/services/am/IAllSystemAppletProxiesService.cpp
        ${source_DIR}/skyline/services/am/IApplicationProxyService.cpp
        ${source_DIR}/skyline/services/am/proxy/base_proxy.cpp
        ${source_DIR}/skyline/services/am/proxy/IApplicationProxy.cpp
        ${source_DIR}/skyline/services/am/proxy/ILibraryAppletProxy.cpp
        ${source_DIR}/skyline/services/am/proxy/IOverlayAppletProxy.cpp
        ${source_DIR}/skyline/services/am/proxy/ISystemAppletProxy.cpp
        ${source_DIR}/skyline/services/am/controller/IAppletCommonFunctions.cpp
        ${source_DIR}/skyline/services/am/controller/IApplicationFunctions.cpp
        ${source_DIR}/skyline/services/am/controller/IAudioController.cpp
        ${source_DIR}/skyline/services/am/controller/ICommonStateGetter.cpp
        ${source_DIR}/skyline/services/am/controller/IDebugFunctions.cpp
        ${source_DIR}/skyline/services/am/controller/IDisplayController.cpp
        ${source_DIR}/skyline/services/am/controller/ILibraryAppletCreator.cpp
        ${source_DIR}/skyline/services/am/controller/ISelfController.cpp
        ${source_DIR}/skyline/services/am/controller/IWindowController.cpp
        ${source_DIR}/skyline/services/am/storage/IStorage.cpp
        ${source_DIR}/skyline/services/am/storage/IStorageAccessor.cpp
        ${source_DIR}/skyline/services/am/applet/ILibraryAppletAccessor.cpp
        ${source_DIR}/skyline/services/codec/IHardwareOpusDecoder.cpp
        ${source_DIR}/skyline/services/codec/IHardwareOpusDecoderManager.cpp
        ${source_DIR}/skyline/services/hid/IHidServer.cpp
        ${source_DIR}/skyline/services/hid/IAppletResource.cpp
        ${source_DIR}/skyline/services/hid/IActiveVibrationDeviceList.cpp
        ${source_DIR}/skyline/services/timesrv/common.cpp
        ${source_DIR}/skyline/services/timesrv/core.cpp
        ${source_DIR}/skyline/services/timesrv/time_shared_memory.cpp
        ${source_DIR}/skyline/services/timesrv/timezone_manager.cpp
        ${source_DIR}/skyline/services/timesrv/time_manager_server.cpp
        ${source_DIR}/skyline/services/timesrv/IStaticService.cpp
        ${source_DIR}/skyline/services/timesrv/ISystemClock.cpp
        ${source_DIR}/skyline/services/timesrv/ISteadyClock.cpp
        ${source_DIR}/skyline/services/timesrv/ITimeZoneService.cpp
        ${source_DIR}/skyline/services/glue/IStaticService.cpp
        ${source_DIR}/skyline/services/glue/ITimeZoneService.cpp
        ${source_DIR}/skyline/services/fssrv/IFileSystemProxy.cpp
        ${source_DIR}/skyline/services/fssrv/IFileSystem.cpp
        ${source_DIR}/skyline/services/fssrv/IFile.cpp
        ${source_DIR}/skyline/services/fssrv/IStorage.cpp
        ${source_DIR}/skyline/services/fssrv/IDirectory.cpp
        ${source_DIR}/skyline/services/nvdrv/INvDrvServices.cpp
        ${source_DIR}/skyline/services/nvdrv/driver.cpp
        ${source_DIR}/skyline/services/nvdrv/core/nvmap.cpp
        ${source_DIR}/skyline/services/nvdrv/core/syncpoint_manager.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvdevice.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvmap.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvhost/as_gpu.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvhost/ctrl.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvhost/ctrl_gpu.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvhost/gpu_channel.cpp
        ${source_DIR}/skyline/services/nvdrv/devices/nvhost/host1x_channel.cpp
        ${source_DIR}/skyline/services/hosbinder/parcel.cpp
        ${source_DIR}/skyline/services/hosbinder/IHOSBinderDriver.cpp
        ${source_DIR}/skyline/services/hosbinder/GraphicBufferProducer.cpp
        ${source_DIR}/skyline/services/visrv/IDisplayService.cpp
        ${source_DIR}/skyline/services/visrv/IApplicationDisplayService.cpp
        ${source_DIR}/skyline/services/visrv/IManagerDisplayService.cpp
        ${source_DIR}/skyline/services/visrv/IRootService.cpp
        ${source_DIR}/skyline/services/visrv/ISystemDisplayService.cpp
        ${source_DIR}/skyline/services/pl/IPlatformServiceManager.cpp
        ${source_DIR}/skyline/services/aocsrv/IAddOnContentManager.cpp
        ${source_DIR}/skyline/services/pctl/IParentalControlServiceFactory.cpp
        ${source_DIR}/skyline/services/pctl/IParentalControlService.cpp
        ${source_DIR}/skyline/services/lm/ILogService.cpp
        ${source_DIR}/skyline/services/lm/ILogger.cpp
        ${source_DIR}/skyline/services/account/IAccountServiceForApplication.cpp
        ${source_DIR}/skyline/services/account/IManagerForApplication.cpp
        ${source_DIR}/skyline/services/account/IProfile.cpp
        ${source_DIR}/skyline/services/friends/IServiceCreator.cpp
        ${source_DIR}/skyline/services/friends/IFriendService.cpp
        ${source_DIR}/skyline/services/friends/INotificationService.cpp
        ${source_DIR}/skyline/services/nfp/IUserManager.cpp
        ${source_DIR}/skyline/services/nfp/IUser.cpp
        ${source_DIR}/skyline/services/nifm/IStaticService.cpp
        ${source_DIR}/skyline/services/nifm/IGeneralService.cpp
        ${source_DIR}/skyline/services/nifm/IRequest.cpp
        ${source_DIR}/skyline/services/socket/bsd/IClient.cpp
        ${source_DIR}/skyline/services/spl/IRandomInterface.cpp
        ${source_DIR}/skyline/services/ssl/ISslService.cpp
        ${source_DIR}/skyline/services/ssl/ISslContext.cpp
        ${source_DIR}/skyline/services/prepo/IPrepoService.cpp
        ${source_DIR}/skyline/services/mmnv/IRequest.cpp
        )
target_include_directories(skyline PRIVATE ${source_DIR}/skyline)
# target_precompile_headers(skyline PRIVATE ${source_DIR}/skyline/common.h) # PCH will currently break Intellisense
target_compile_options(skyline PRIVATE -Wall -Wno-unknown-attributes -Wno-c++20-extensions -Wno-c++17-extensions -Wno-c99-designator -Wno-reorder -Wno-missing-braces -Wno-unused-variable -Wno-unused-private-field -Wno-dangling-else -Wconversion)

# Include headers from libraries as system headers to silence warnings from them
function(target_link_libraries_system target)
    set(libraries ${ARGN})
    foreach (library ${libraries})
        if (TARGET ${library})
            get_target_property(library_include_directories ${library} INTERFACE_INCLUDE_DIRECTORIES)
            if (NOT "${library_include_directories}" STREQUAL "library_include_directories-NOTFOUND")
                target_include_directories(${target} SYSTEM PRIVATE ${library_include_directories})
            endif ()
        endif ()
        target_link_libraries(${target} ${library})
    endforeach (library)
endfunction(target_link_libraries_system)

target_link_libraries_system(skyline android perfetto fmt lz4_static tzcode oboe vkma mbedcrypto opus Boost::container)
