set (libname "bacnetmstp")
set (libdescription "upm driver module for BACnet MS/TP devices")
set (module_src ${libname}.cxx device-client.c)
set (module_hpp ${libname}.hpp)

pkg_search_module(BACNET libbacnet)
if (BACNET_FOUND)
  set (reqlibname "libbacnet")
  include_directories(${BACNET_INCLUDE_DIRS})
  upm_module_init()
  add_dependencies(${libname} ${BACNET_LIBRARIES})
  target_link_libraries(${libname} ${BACNET_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
  if (BUILDSWIG)
    if (BUILDSWIGNODE)
      swig_link_libraries (jsupm_${libname} ${BACNET_LIBRARIES} ${MRAA_LIBRARIES} ${NODE_LIBRARIES})
    endif()
    if (BUILDSWIGPYTHON)
      swig_link_libraries (pyupm_${libname} ${BACNET_LIBRARIES} ${PYTHON_LIBRARIES} ${MRAA_LIBRARIES})
    endif()
    if (BUILDSWIGJAVA)
        swig_link_libraries (javaupm_${libname} ${BACNET_LIBRARIES} ${MRAAJAVA_LDFLAGS} ${JAVA_LDFLAGS})
    endif()
  endif()
endif ()
