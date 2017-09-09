# Chibi-scheme CMakeLists.txt
#
#  INPUT: 
#    CHIBI_SCHEME_ROOT : fullpath to chibi-scheme repository root

cmake_minimum_required(VERSION 2.8.7)
project(chibi-scheme)

#
# Features
#

add_definitions(
    # Disable green threads: It depends on non-blocking I/O
    -DSEXP_USE_GREEN_THREADS=0 

    -DSEXP_USE_DL=0
    -DBUILDING_DLL)

set(chibi-scheme-exclude-modules-win32
    lib/chibi/filesystem.sld
    lib/chibi/net.sld
    lib/chibi/process.sld
    lib/chibi/stty.sld
    lib/chibi/system.sld
    lib/chibi/time.sld)

#
# Sources
#

set(chibi-scheme-srcs
    # SEXP
    gc.c
    sexp.c
    bignum.c
    gc_heap.c

    # Eval
    opcodes.c
    vm.c
    eval.c
    simplify.c
    )

# Update source list using CHIBI_SCHEME_ROOT

foreach(e ${chibi-scheme-srcs})
    list(APPEND tmp ${CHIBI_SCHEME_ROOT}/${e})
endforeach()

set(chibi-scheme-srcs ${tmp})

include_directories(
    ${CHIBI_SCHEME_ROOT}/include
    ${CMAKE_CURRENT_BINARY_DIR}/include)

#
# Bootstrap
#

add_executable(chibi-scheme-bootstrap
    ${chibi-scheme-srcs}
    ${CHIBI_SCHEME_ROOT}/main.c)

if(WIN32)
    target_link_libraries(chibi-scheme-bootstrap ws2_32 shlwapi)
endif()

# FIXME: I cannot recall how do we properly do this...
#if(CYGWIN OR WIN32)
#    set(soext "dll")
#else()
#    set(soext "so")
#endif()

set(soext "NEVERLAND") # Prevent .so unexpectedly loaded

#
# Generate modules
#

# FIXME: Currently, it depends on GLOB thus we have to re-run CMake
#        when we've gotten additional/removed library 

file(GLOB_RECURSE stubs RELATIVE ${CHIBI_SCHEME_ROOT}/lib
    ${CHIBI_SCHEME_ROOT}/lib/*.stub)

file(GLOB_RECURSE slds RELATIVE ${CHIBI_SCHEME_ROOT}
    ${CHIBI_SCHEME_ROOT}/lib/*.sld)

if(WIN32)
    list(REMOVE_ITEM slds 
        ${chibi-scheme-exclude-modules-win32})
endif()

set(chibi-ffi ${CHIBI_SCHEME_ROOT}/tools/chibi-ffi)
set(chibi-genstatic ${CHIBI_SCHEME_ROOT}/tools/chibi-genstatic)

set(stuboutdir ${CMAKE_CURRENT_BINARY_DIR}/stubs/lib)
foreach(e ${stubs})
    get_filename_component(stubdir ${e} PATH)
    get_filename_component(basename ${e} NAME_WE)
    set(stubfile ${CHIBI_SCHEME_ROOT}/lib/${e})
    set(stubdir ${stuboutdir}/${stubdir})
    set(stubout ${stubdir}/${basename}.c)
    file(MAKE_DIRECTORY ${stubdir})
    add_custom_command(OUTPUT ${stubout}
        COMMAND chibi-scheme-bootstrap
        ${chibi-ffi} ${stubfile} ${stubout}
        DEPENDS ${stubfile} ${chibi-ffi}
        WORKING_DIRECTORY ${CHIBI_SCHEME_ROOT})
    list(APPEND stubouts ${stubout})
endforeach()
add_custom_target(chibi-scheme-stubs DEPENDS ${stubouts})

string(REPLACE ";" "\n" genstatic-input "${slds}")
set(clibin ${CMAKE_CURRENT_BINARY_DIR}/clib-in.txt)
set(clibout ${CMAKE_CURRENT_BINARY_DIR}/clib.c)
set(genstatic-helper ${CMAKE_CURRENT_LIST_DIR}/chibi-genstatic-helper.cmake)
file(WRITE ${clibin} "${genstatic-input}")
message(STATUS "${slds}")

add_custom_command(OUTPUT ${clibout}
    COMMAND 
    ${CMAKE_COMMAND} 
    -DEXEC=$<TARGET_FILE:chibi-scheme-bootstrap>
    -DGENSTATIC=${chibi-genstatic}
    -DSTUBS=${clibin}
    -DOUT=${clibout}
    -P ${genstatic-helper}
    WORKING_DIRECTORY ${CHIBI_SCHEME_ROOT}
    DEPENDS 
    chibi-scheme-bootstrap
    ${chibi-genstatic}
    ${genstatic-helper}
    ${slds})

#
# Interpreter
#

include_directories(
    ${stuboutdir}/.. # Strip "lib"
    ${CHIBI_SCHEME_ROOT})
add_executable(chibi-scheme
    ${chibi-scheme-srcs}
    ${clibout}
    ${CHIBI_SCHEME_ROOT}/main.c)

set_target_properties(chibi-scheme
    PROPERTIES COMPILE_DEFINITIONS "SEXP_USE_STATIC_LIBS=1")

add_dependencies(chibi-scheme chibi-scheme-stubs)

if(WIN32)
    target_link_libraries(chibi-scheme ws2_32 shlwapi)
endif()

#
# Generate "chibi/install.h" 
#

# FIXME: I cannot recall how do we properly do this...
if(CYGWIN OR WIN32)
    set(thePrefix "bin")
else()
    set(thePrefix "lib")
endif()

# Select path separator
if(WIN32)
    set(pathsep "\\;")
else()
    set(pathsep ":")
endif()

# Select platform
if(WIN32)
    set(platform "windows")
else()
    set(platform "unknown")
endif()

file(READ ${CHIBI_SCHEME_ROOT}/RELEASE release)
string(STRIP ${release} release)

file(READ ${CHIBI_SCHEME_ROOT}/VERSION version)
string(STRIP ${version} version)
set(version "${version}-cmake")

set(default_module_path
    ""
    #"${CMAKE_INSTALL_PREFIX}/${thePrefix}${pathsep}${CMAKE_INSTALL_PREFIX}/bin"
    )

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/include/chibi)

file(WRITE
    ${CMAKE_CURRENT_BINARY_DIR}/include/chibi/install.h
    "#define sexp_so_extension \"${soext}\"
#define sexp_default_module_path \"${default_module_path}\"
#define sexp_platform \"${platform}\"
#define sexp_version \"\"
#define sexp_release_name \"${release}\"")

# 
# Testing
#

enable_testing()

set(coretests
    r7rs-tests
    division-tests
    r5rs-tests
    syntax-tests
    unicode-tests
    ## Require threads
    # lib-tests
    )

foreach(e ${coretests})
    add_test(NAME "core-${e}"
        COMMAND chibi-scheme tests/${e}.scm
        WORKING_DIRECTORY ${CHIBI_SCHEME_ROOT})
endforeach()
