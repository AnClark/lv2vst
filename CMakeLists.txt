cmake_minimum_required(VERSION 3.10)

project(lv2vst
  VERSION 0.1.0
  DESCRIPTION "Expose LV2 plugins as VST2 plugins to a VST plugin-host on Windows, OSX and Linux"
  LANGUAGES C CXX)

set (PACKAGE_NAME ${PROJECT_NAME})
set (PACKAGE_VERSION "${VERSION}")
set (PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set (PACKAGE_TARNAME ${PROJECT_NAME})

set (CONFIG_VST_DIR_WINDOWS "C:/Program Files/Steinberg/VSTPlugins" CACHE STRING "Specify VST plugin install path on Windows.")
set (CONFIG_VST_DIR_UNIX ${CMAKE_INSTALL_PREFIX} CACHE STRING "Specify VST plugin install path on Unix (Linux or macOS).")


set (PLUGIN_SRC
    src/instantiate.cc
    src/loadlib.cc
    src/lv2ttl.cc
    src/lv2vst.cc
    src/lv2vstui.cc
    src/state.cc
    src/vstmain.cc
    src/worker.cc
)

set (PLUGIN_HEADERS
    src/loadlib.h
    src/lv2desc.h
    src/lv2vst.h
    src/lv2ttl.h
    src/ringbuffer.h
    src/shell.h
    src/uri_map.h
    src/vst.h
    src/worker.h
)

set (LV2_SRC
    lib/lilv/collections.c
    lib/lilv/instance.c
    lib/lilv/lib.c
    lib/lilv/node.c
    lib/lilv/plugin.c
    lib/lilv/pluginclass.c
    lib/lilv/port.c
    lib/lilv/query.c
    lib/lilv/scalepoint.c
    lib/lilv/state.c
    lib/lilv/ui.c
    lib/lilv/util.c
    lib/lilv/world.c
    lib/lilv/zix/tree.c
    lib/serd/env.c
    lib/serd/node.c
    lib/serd/reader.c
    lib/serd/string.c
    lib/serd/uri.c
    lib/serd/writer.c
    lib/sord/sord.c
    lib/sord/syntax.c
    lib/sord/zix/btree.c
    lib/sord/zix/digest.c
    lib/sord/zix/hash.c
    lib/sratom/sratom.c
)

set (INCLUDES
    include/vestige.h
    include/lilv_config.h
    include/lilv/lilv.h
    include/serd_config.h
    include/serd/serd.h
    include/sord_config.h
    include/sord/sord.h
    include/sratom/sratom.h
    include/lv2/lv2plug.in
)

# TODO: Allow user to specify BUNDLES and WHITELIST here (line:123)

set (PTHREAD_CFLAGS "-D__CLEANUP_C -DPTW32_BUILD_INLINED -DPTW32_STATIC_LIB -DHAVE_PTW32_CONFIG_H")
add_library (pthread_w32 STATIC
  lib/pthreads-w32/pthread.c
)
target_include_directories (pthread_w32 PRIVATE "lib/pthreads-w32/")
target_compile_options(pthread_w32 PRIVATE ${PTHREAD_CFLAGS})
target_compile_options(pthread_w32 PRIVATE -fvisibility=hidden -mstackrealign -O3)


if (APPLE)
  set (STRIP_FLAGS -x)
  set (LIB_EXT .dylib)

  set (VSTLDFLAGS -dynamiclib)
  set (LOADLIBES -lm -ldl)

  set (PLATFORM_CXXFLAGS -Wno-deprecated-declarations -fvisibility=hidden -fvisibility-inlines-hidden -fdata-sections -ffunction-sections -fPIC -pthread)
  set (PLATFORM_LDFLAGS -headerpad_max_install_names -Bsymbolic)

  if (CMAKE_BUILD_TYPE MATCHES "debug")
    set (PLATFORM_LDFLAGS ${PLATFORM_LDFLAGS} -fvisibility=hidden -fdata-sections -ffunction-sections -Wl,-dead_strip)
  endif ()

elseif (WIN32)
  set (STRIP_FLAGS -s)
  set (LIB_EXT .dll)

  set (VSTLDFLAGS -Wl,-Bstatic -Wl,-Bdynamic -Wl,--as-needed -shared)
  set (LOADLIBES -lm -lws2_32)

  set (PLATFORM_CXXFLAGS -fvisibility=hidden -fdata-sections -ffunction-sections -mfpmath=sse)
  set (PLATFORM_LDFLAGS -static-libgcc -static-libstdc++)

  if (CMAKE_BUILD_TYPE MATCHES "debug")
    set (PLATFORM_LDFLAGS ${PLATFORM_LDFLAGS} -Wl,--gc-sections -Wl,-O1 -Wl,--as-needed -Wl,--strip-all)
  endif ()

  set (PLATFORM_CXXFLAGS ${PLATFORM_CXXFLAGS} -mstackrealign ${PTHREAD_FLAGS})

else ()
  set (STRIP_FLAGS -s)
  set (LIB_EXT .so)

  set (VSTLDFLAGS -Wl,-Bstatic -Wl,-Bdynamic -Wl,--as-needed -shared)
  set (LOADLIBES -lm -ldl)

  set (PLATFORM_CXXFLAGS -fvisibility=hidden -fdata-sections -ffunction-sections -mfpmath=sse -fPIC -pthread)
  set (PLATFORM_LDFLAGS -static-libgcc -static-libstdc++)

  if (CMAKE_BUILD_TYPE MATCHES "debug")
    set (PLATFORM_LDFLAGS ${PLATFORM_LDFLAGS} -Wl,--gc-sections -Wl,-O1 -Wl,--as-needed -Wl,--strip-all)
  endif ()
endif ()


add_library (${PROJECT_NAME} SHARED
  ${INCLUDES}
  ${PLUGIN_HEADERS}
  ${PLUGIN_SRC}
  ${LV2_SRC}
)
target_include_directories (${PROJECT_NAME} PRIVATE include/)
target_include_directories (${PROJECT_NAME} PRIVATE lib/sord/)
target_include_directories (${PROJECT_NAME} PRIVATE lib/lilv/)
target_include_directories (${PROJECT_NAME} PRIVATE "lib/pthreads-w32/")
target_compile_options (${PROJECT_NAME} PRIVATE ${PLATFORM_CXXFLAGS})
target_link_options (${PROJECT_NAME} PRIVATE ${VSTLDFLAGS} ${PLATFORM_LDFLAGS})
target_link_libraries(${PROJECT_NAME} PUBLIC pthread_w32 ${LOADLIBES})


if (CMAKE_BUILD_TYPE MATCHES "debug")
  target_compile_options (${PROJECT_NAME} PRIVATE -g -O0)
else ()
  target_compile_options (${PROJECT_NAME} PRIVATE -msse -msse2 -ffast-math -fomit-frame-pointer -O3 -fno-finite-math-only -DNDEBUG)
  add_custom_command(TARGET ${PROJECT_NAME}  POST_BUILD
    COMMAND strip ${STRIP_FLAGS} lib${PROJECT_NAME}${LIB_EXT})
endif ()


if (WIN32)
  install (FILES ${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}${LIB_EXT}
    RENAME ${PROJECT_NAME}${LIB_EXT}
    DESTINATION ${CONFIG_VST_DIR_WINDOWS})
else ()
  install (FILES ${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}${LIB_EXT}
    RENAME ${PROJECT_NAME}${LIB_EXT}
    DESTINATION ${CONFIG_VST_DIR_UNIX})
endif ()
