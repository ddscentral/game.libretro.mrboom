cmake_minimum_required(VERSION 3.5)
project(mingw)

if (0)
  foreach(repo msys mingw32 mingw64)
    if(${repo} STREQUAL msys)
      file(WRITE ${CMAKE_CURRENT_SOURCE_DIR}/etc/pacman.d/mirrorlist.${repo} "Server = http://mirrors.kodi.tv/build-deps/win32/msys2/repos/${repo}2/$arch\n")
    else()
      file(WRITE ${CMAKE_CURRENT_SOURCE_DIR}/etc/pacman.d/mirrorlist.${repo} "Server = http://mirrors.kodi.tv/build-deps/win32/msys2/repos/${repo}\n")
    endif()
  endforeach()
endif()

include(CheckSymbolExists)
check_symbol_exists(_X86_ "Windows.h" _X86_)
check_symbol_exists(_AMD64_ "Windows.h" _AMD64_)
check_symbol_exists(_M_ARM64 "Windows.h" _M_ARM64)

if(_X86_)
   set(HOST mingw-w64-i686)
   set(MINGW_PATH "mingw32")
elseif(_AMD64_)
   set(HOST mingw-w64-x86_64)
   set(MINGW_PATH "mingw64")
elseif(_M_ARM64)
   set(HOST mingw-w64-clang-aarch64)
   set(MINGW_PATH "clangarm64")
else()
   message(FATAL_ERROR "Unsupported architecture")
endif()

execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "pacman-key --init" ERROR_QUIET)
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "pacman-key --populate" ERROR_QUIET)
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "rm /var/lib/pacman/sync/*")
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "sed -i s/^XferCommand/#sXferCommand/g /etc/pacman.conf")
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "echo n | pacman -Syuu")
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "pacman --noconfirm -S make ${HOST}-gcc")
execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe --login -c "sed -i s/^#sXferCommand/XferCommand/g /etc/pacman.conf")

#if($ENV{PROCESSOR_ARCHITECTURE} MATCHES "ARM64")
   # mingw-w64-cross-mingwarm64 does not create gcc anywhere in the path
   #execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe -lc
   #"which gcc || find /opt/bin -type f -name 'aarch64-w64-mingw32-*' -exec /bin/bash -c 'ln -s {} $(echo /opt/bin/{} | sed -e ''s#/opt/bin/aarch64-w64-mingw32-##g'')' \\;")
   #execute_process(COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/usr/bin/bash.exe -lc
   #"which cc || ln -s /opt/bin/gcc /opt/bin/cc")
#endif()

#if(NOT $ENV{PROCESSOR_ARCHITECTURE} MATCHES "ARM64")
   file(GLOB_RECURSE shared_libs ${CMAKE_CURRENT_SOURCE_DIR}/${MINGW_PATH}/*.dll.a)
   file(REMOVE ${shared_libs})
#endif()

configure_file(MinGWConfig.cmake.in MinGWConfig.cmake @ONLY)
configure_file(make.bat.in ${CMAKE_CURRENT_SOURCE_DIR}/${MINGW_PATH}/bin/make.bat @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/MinGWConfig.cmake DESTINATION ${CMAKE_INSTALL_PREFIX})
