#
# project: fips-libuv
# Cross-platform asynchronous I/O library
#
cmake_minimum_required(VERSION 3.21)

project(fips-libuv)

get_filename_component(FIPS_ROOT_DIR "../fips" ABSOLUTE)
include("${FIPS_ROOT_DIR}/cmake/fips.cmake")

fips_setup()

fips_begin_lib(libuv)

    # Common source files for all platforms
    fips_dir(src)
    fips_files(
        fs-poll.c
        idna.c
        inet.c
        random.c
        strscpy.c
        strtok.c
        thread-common.c
        threadpool.c
        timer.c
        uv-common.c
        uv-data-getter-setters.c
        version.c
    )

    # Windows-specific source files
    if (FIPS_WINDOWS)
        fips_dir(src/win)
        fips_files(
            async.c
            core.c
            detect-wakeup.c
            dl.c
            error.c
            fs.c
            fs-event.c
            getaddrinfo.c
            getnameinfo.c
            handle.c
            loop-watcher.c
            pipe.c
            thread.c
            poll.c
            process.c
            process-stdio.c
            signal.c
            snprintf.c
            stream.c
            tcp.c
            tty.c
            udp.c
            util.c
            winapi.c
            winsock.c
        )
    endif()

    # Unix-specific source files
    if (FIPS_LINUX OR FIPS_MACOS OR FIPS_IOS OR FIPS_ANDROID)
        fips_dir(src/unix)
        fips_files(
            async.c
            core.c
            dl.c
            fs.c
            getaddrinfo.c
            getnameinfo.c
            loop-watcher.c
            loop.c
            pipe.c
            poll.c
            process.c
            random-devurandom.c
            signal.c
            stream.c
            tcp.c
            thread.c
            tty.c
            udp.c
            proctitle.c
        )
    endif()

    # Linux-specific
    if (FIPS_LINUX)
        fips_dir(src/unix)
        fips_files(
            linux.c
            procfs-exepath.c
            random-getrandom.c
            random-sysctl-linux.c
        )
    endif()

    # macOS/iOS specific
    if (FIPS_MACOS OR FIPS_IOS)
        fips_dir(src/unix)
        fips_files(
            bsd-ifaddrs.c
            darwin.c
            darwin-proctitle.c
            fsevents.c
            kqueue.c
            random-getentropy.c
        )
    endif()

    # Android-specific
    if (FIPS_ANDROID)
        fips_dir(src/unix)
        fips_files(
            linux.c
            procfs-exepath.c
            random-getentropy.c
            random-getrandom.c
            random-sysctl-linux.c
        )
    endif()

fips_end_lib()

# Include directories
target_include_directories(libuv
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Platform-specific compile definitions
if (FIPS_WINDOWS)
    target_compile_definitions(libuv PRIVATE
        WIN32_LEAN_AND_MEAN
        _WIN32_WINNT=0x0A00
        _CRT_DECLARE_NONSTDC_NAMES=0
    )
    target_link_libraries(libuv
        psapi
        user32
        advapi32
        iphlpapi
        userenv
        ws2_32
        dbghelp
        ole32
        shell32
    )
else()
    target_compile_definitions(libuv PRIVATE
        _FILE_OFFSET_BITS=64
        _LARGEFILE_SOURCE
    )
endif()

if (FIPS_LINUX)
    target_compile_definitions(libuv PRIVATE _GNU_SOURCE _POSIX_C_SOURCE=200112)
    target_link_libraries(libuv dl rt m)
endif()

if (FIPS_MACOS OR FIPS_IOS)
    target_compile_definitions(libuv PRIVATE
        _DARWIN_UNLIMITED_SELECT=1
        _DARWIN_USE_64_BIT_INODE=1
    )
    target_link_libraries(libuv m)
endif()

if (FIPS_ANDROID)
    target_compile_definitions(libuv PRIVATE _GNU_SOURCE)
    target_link_libraries(libuv dl)
endif()

# Threading
if (NOT FIPS_WINDOWS AND NOT FIPS_ANDROID)
    find_package(Threads REQUIRED)
    target_link_libraries(libuv Threads::Threads)
endif()

# C11 standard
set_target_properties(libuv PROPERTIES C_STANDARD 11)
set_target_properties(libuv PROPERTIES C_STANDARD_REQUIRED ON)
set_target_properties(libuv PROPERTIES C_EXTENSIONS ON)

# Disable warnings for third-party code
if (FIPS_MSVC)
    target_compile_options(libuv PRIVATE
        /wd4100  # unreferenced formal parameter
        /wd4127  # conditional expression is constant
        /wd4201  # nonstandard extension
        /wd4206  # nonstandard extension: empty translation unit
        /wd4210  # nonstandard extension: function given file scope
        /wd4232  # nonstandard extension: dllimport address
        /wd4456  # declaration hides local
        /wd4457  # declaration hides function parameter
        /wd4459  # declaration hides global
        /wd4706  # assignment within conditional
        /wd4996  # deprecated function
        /utf-8
    )
elseif (FIPS_CLANG OR FIPS_GCC)
    target_compile_options(libuv PRIVATE
        -Wno-unused-parameter
        -fno-strict-aliasing
    )
endif()

# Create alias for compatibility with find_package(libuv)
add_library(libuv::uv_a ALIAS libuv)
