macro (file_to_list readfile outlist)
  FILE(READ "${readfile}" contents)
  STRING(REGEX REPLACE ";" "\\\\;" contents "${contents}")
  STRING(REGEX REPLACE "\n" ";" contents "${contents}")
  set("${outlist}" "${contents}" )
endmacro()

file_to_list ("javaswig_blacklist" JAVASWIG_BLACKLIST)
file_to_list ("pythonswig_blacklist" PYTHONSWIG_BLACKLIST)
file_to_list ("nodeswig_blacklist" NODESWIG_BLACKLIST)

macro(subdirlist result curdir)
  file(GLOB children RELATIVE ${curdir} ${curdir}/*)
  set(dirlist "")
  foreach(child ${children})
      if(IS_DIRECTORY ${curdir}/${child})
        set(dirlist ${dirlist} ${child})
    endif()
  endforeach()
  set(${result} ${dirlist})
endmacro()

macro (upm_CREATE_INSTALL_PKGCONFIG generated_file install_location)
  configure_file (${PROJECT_SOURCE_DIR}/src/pkgconfig.in
    ${CMAKE_CURRENT_BINARY_DIR}/${generated_file} @ONLY)
  install (FILES ${CMAKE_CURRENT_BINARY_DIR}/${generated_file} DESTINATION ${install_location})
endmacro (upm_CREATE_INSTALL_PKGCONFIG)

macro(upm_SWIG_PYTHON)
  if (BUILDSWIGPYTHON AND BUILDSWIG)

    include_directories (
      ${CMAKE_CURRENT_SOURCE_DIR}/..
    )

    set_source_files_properties (pyupm_${libname}.i PROPERTIES CPLUSPLUS ON)
    set_source_files_properties (pyupm_${libname}.i PROPERTIES SWIG_FLAGS "-I${CMAKE_CURRENT_BINARY_DIR}/..")
    swig_add_module (pyupm_${libname} python pyupm_${libname}.i ${module_src})
    swig_link_libraries (pyupm_${libname} ${PYTHON_LIBRARIES} ${MRAA_LIBRARIES})
    target_include_directories ( ${SWIG_MODULE_pyupm_${libname}_REAL_NAME}
      PUBLIC
      "${PYTHON_INCLUDE_PATH}"
      "${PYTHON_INCLUDE_DIRS}"
     )
    install (FILES ${CMAKE_CURRENT_BINARY_DIR}/_pyupm_${libname}.so
         ${CMAKE_CURRENT_BINARY_DIR}/pyupm_${libname}.py
         DESTINATION ${LIB_INSTALL_DIR}/python${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}/site-packages/
         COMPONENT ${libname})
  endif()
endmacro()

macro(upm_SWIG_NODE)
  if (BUILDSWIGNODE AND BUILDSWIG)
    # SWIG treats SWIG_FLAGS as a list and not a string so semicolon seperation is
    # required. This hardcodes V8_VERSION to be <10 but I assume that's not going
    # to be a problem for a little while! SWIG uses a padded SWIG_V8 version which
    # we hack together from our findnode module.
    set (V8_VERSION_HEX 0x0${V8_VERSION_MAJOR}${V8_VERSION_MINOR}${V8_VERSION_PATCH})
    string (LENGTH "${V8_VERSION_HEX}" V8_VERSION_HEX_length)
    while (V8_VERSION_HEX_length LESS 8)
      set (V8_VERSION_HEX "${V8_VERSION_HEX}0")
      string (LENGTH "${V8_VERSION_HEX}" V8_VERSION_HEX_length)
    endwhile ()

    include_directories (
      ${CMAKE_CURRENT_SOURCE_DIR}/..
    )

    set_property (SOURCE jsupm_${libname}.i PROPERTY SWIG_FLAGS "-node" "-DV8_VERSION=${V8_VERSION_HEX}")
    set_source_files_properties (jsupm_${libname}.i PROPERTIES CPLUSPLUS ON)
    swig_add_module (jsupm_${libname} javascript jsupm_${libname}.i ${module_src})
    swig_link_libraries (jsupm_${libname} ${MRAA_LIBRARIES} ${NODE_LIBRARIES})
    target_include_directories ( ${SWIG_MODULE_jsupm_${libname}_REAL_NAME}
      PUBLIC
      "${NODE_INCLUDE_DIRS}"
    )

    set_target_properties (jsupm_${libname} PROPERTIES
      COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -DBUILDING_NODE_EXTENSION -DSWIG_V8_VERSION=${V8_VERSION_HEX}"
      PREFIX ""
      SUFFIX ".node"
    )
    if (${V8_VERSION_MAJOR} GREATER 3)
      if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.7")
          message(FATAL_ERROR " **ERROR** GCC 4.7 or above is required to compile jsupm_${libname} ")
        endif()
      endif()

      if (NOT ENABLECXX11)
          set_property (TARGET jsupm_${libname} PROPERTY CXX_STANDARD 11)
          set_property (TARGET jsupm_${libname} PROPERTY CXX_STANDARD_REQUIRED ON)

          if (CMAKE_VERSION VERSION_LESS "3.1")
            if (CMAKE_COMPILER_IS_GNUCXX)
              if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.7")
                message (FATAL_ERROR " FATAL ERROR: GNU gcc compiler is also too old (need 4.7+, but ${CMAKE_CXX_COMPILER_VERSION}) and does not support C++11 standard.")
              endif ()
              set (UPM_CXX11_WORKAROUND_OPTION "-std=gnu++11")
            else ()
              set (UPM_CXX11_WORKAROUND_OPTION "-std=c++11")
            endif ()
            set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${UPM_CXX11_WORKAROUND_OPTION} ")
          endif ()
      endif ()

    endif()
    createpackagejson(${libname})
    install (FILES ${CMAKE_CURRENT_BINARY_DIR}/jsupm_${libname}.node
         DESTINATION ${NODE_MODULE_INSTALL_PATH} COMPONENT ${libname})
  endif()
endmacro()

macro(upm_SWIG_JAVA)
  if (BUILDSWIGJAVA AND BUILDSWIG)

    FIND_PACKAGE (JNI REQUIRED)
    pkg_check_modules (MRAAJAVA REQUIRED mraajava>=0.8.0)

    include_directories (
      ${JAVA_INCLUDE_PATH}
      ${JAVA_INCLUDE_PATH2}
      ${CMAKE_CURRENT_SOURCE_DIR}/..
    )

    set_source_files_properties (javaupm_${libname}.i PROPERTIES CPLUSPLUS ON)
    set_source_files_properties (javaupm_${libname}.i PROPERTIES SWIG_FLAGS ";-package;upm_${libname};-I${CMAKE_BINARY_DIR}/src")
    swig_add_module (javaupm_${libname} java javaupm_${libname}.i ${module_src})
    swig_link_libraries (javaupm_${libname} ${MRAAJAVA_LDFLAGS} ${JAVA_LDFLAGS})
    target_include_directories ( ${SWIG_MODULE_javaupm_${libname}_REAL_NAME}
      PUBLIC
      "${JAVA_INCLUDE_DIRS}"
      "${JAVA_INCLUDE_PATH}"
     )
    set_target_properties (javaupm_${libname} PROPERTIES
      COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -fpermissive -DJAVACALLBACK"
      PREFIX "lib"
      SUFFIX ".so"
    )
    install (TARGETS javaupm_${libname} LIBRARY DESTINATION ${LIB_INSTALL_DIR})
    install (FILES ${CMAKE_CURRENT_BINARY_DIR}/upm_${libname}.jar DESTINATION ${LIB_INSTALL_DIR}/../lib/java)

    if (NOT DEFINED $ENV{JAVA_HOME_NATIVE})
        set (JAVAC $ENV{JAVA_HOME}/bin/javac)
        set (JAR $ENV{JAVA_HOME}/bin/jar)
    else ()
        set (JAVAC $ENV{JAVA_HOME_NATIVE}/bin/javac)
        set (JAR $ENV{JAVA_HOME_NATIVE}/bin/jar)
    endif ()

    add_custom_command (TARGET javaupm_${libname}
        POST_BUILD
        COMMAND cmake -E echo "Compiling java.."
        COMMAND cmake -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/upm_${libname}
        COMMAND ${JAVAC} *.java -d ${CMAKE_CURRENT_BINARY_DIR}
        COMMAND cmake -E echo "Creating jar"
        COMMAND ${JAR} cvf upm_${libname}.jar upm_${libname}
    )

  endif()
endmacro()

macro(upm_doxygen)
  if (DOXYGEN_FOUND)
    if(NOT DEFINED classname)
      set (classname ${libname})
    endif()
    set (CMAKE_SWIG_FLAGS -DDOXYGEN=${DOXYGEN_FOUND})
    if (BUILDSWIGPYTHON AND BUILDSWIG)
      add_dependencies (_pyupm_${libname} pyupm_doxy2swig)
      add_dependencies (pydoc _pyupm_${libname})
    else ()
      add_dependencies (${libname} doc)
    endif ()
  endif ()
endmacro()

if (SWIG_FOUND)
  if (BUILDSWIGPYTHON)
    if(NOT PYTHONLIBS_FOUND)
      find_package (PythonLibs ${PYTHONBUILD_VERSION} REQUIRED)
      string (REPLACE "." ";" PYTHON_VERSION_LIST ${PYTHONLIBS_VERSION_STRING})
      list (GET PYTHON_VERSION_LIST 0 PYTHON_VERSION_MAJOR)
      list (GET PYTHON_VERSION_LIST 1 PYTHON_VERSION_MINOR)
    endif()
  endif(BUILDSWIGPYTHON)
  if (BUILDSWIGNODE)
    if(NOT NODE_FOUND)
      find_package(Node)
    endif()
    if(SWIG_VERSION VERSION_LESS 3.0.5 AND NODE_VERSION_STRING VERSION_GREATER 0.12)
      message("WARNING - SWIG 3.0.5+ required for building with nodejs 0.12. Current version is ${SWIG_VERSION}")
    endif()
    find_path (NODE_ROOT_DIR "include/node/node.h")
    set (NODE_INCLUDE_DIRS
      ${NODE_ROOT_DIR}/include/src
      ${NODE_ROOT_DIR}/include/node
      ${NODE_ROOT_DIR}/include/deps/v8/include
      ${NODE_ROOT_DIR}/include/deps/uv/include
    )
    macro(createpackagejson)
      configure_file (${PROJECT_SOURCE_DIR}/src/package.json.in ${CMAKE_CURRENT_BINARY_DIR}/package.json @ONLY)
      # If a CMAKE_INSTALL_PREFIX has NOT been provided, set NODE_MODULE_INSTALL_PATH
      # base on the NODE_ROOT_DIR.
      if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
        set (NODE_MODULE_INSTALL_PATH ${NODE_ROOT_DIR}/lib/node_modules/jsupm_${libname}/)
      # If a CMAKE_INSTALL_PREFIX has been provided, set NODE_MODULE_INSTALL_PATH
      # relative to the provided install directory.
      else ()
        set (NODE_MODULE_INSTALL_PATH ${CMAKE_INSTALL_PREFIX}/lib/node_modules/jsupm_${libname}/)
      endif ()
      install(FILES ${CMAKE_CURRENT_BINARY_DIR}/package.json
          DESTINATION ${NODE_MODULE_INSTALL_PATH} COMPONENT ${libname})
    endmacro()
  endif(BUILDSWIGNODE)
endif()

macro(upm_module_init)
  link_directories (${MRAA_LIBDIR})
  add_library (${libname} SHARED ${module_src})
  foreach (linkflag ${ARGN})
    target_link_libraries (${libname} ${linkflag})
  endforeach ()
  include_directories (${MRAA_INCLUDE_DIRS} . ..)
  target_link_libraries (${libname} ${MRAA_LIBRARIES})
  set_target_properties(
    ${libname}
    PROPERTIES PREFIX "libupm-"
    SOVERSION ${upm_VERSION_MAJOR}
    VERSION ${upm_VERSION_STRING}
  )
  upm_create_install_pkgconfig (upm-${libname}.pc ${LIB_INSTALL_DIR}/pkgconfig)
  if (SWIG_FOUND)
    if (NOT ";${PYTHONSWIG_BLACKLIST};" MATCHES ";${libname};")
      upm_swig_python()
    endif()
    if (NOT ";${NODESWIG_BLACKLIST};" MATCHES ";${libname};")
      upm_swig_node()
    endif()
    if (NOT ";${JAVASWIG_BLACKLIST};" MATCHES ";${libname};")
      upm_swig_java()
    endif()
  endif()
  if (BUILDDOC)
    upm_doxygen()
  endif()
  install(TARGETS ${libname} DESTINATION ${LIB_INSTALL_DIR})
  install (FILES ${module_hpp} DESTINATION include/upm COMPONENT ${libname})

  if (IPK)
    cpack_add_component (${libname} DISPLAY_NAME ${libname} REQUIRED INSTALL_TYPES all)
    set(CPACK_COMPONENT_${libname}_DESCRIPTION "${libdescription}")
  endif()
endmacro(upm_module_init)

# Generate python module documentation from doxygen collateral
if (BUILDDOC AND BUILDSWIGPYTHON AND SWIG_FOUND)
  # doxy2swig the doxygen output
  add_custom_command (OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/pyupm_doxy2swig.i
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/doxy2swig.py
      ${CMAKE_BINARY_DIR}/xml/index.xml
      ${CMAKE_CURRENT_BINARY_DIR}/pyupm_doxy2swig.i
    DEPENDS ${CMAKE_BINARY_DIR}/xml/index.xml
  )
  add_custom_target (pyupm_doxy2swig DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/pyupm_doxy2swig.i)
  add_dependencies (pyupm_doxy2swig doc)
# BUILDDOC not set but still building python modules, generate an empty
# pyupm_doxy2swig.i file (overwriting if necessary)
elseif (BUILDSWIGPYTHON AND SWIG_FOUND)
  message (INFO " Generating empty ${CMAKE_CURRENT_BINARY_DIR}/pyupm_doxy2swig.i")
  file (WRITE ${CMAKE_CURRENT_BINARY_DIR}/pyupm_doxy2swig.i "// Empty doxy2swig stub")
endif (BUILDDOC AND BUILDSWIGPYTHON AND SWIG_FOUND)

if (MODULE_LIST)
  set(SUBDIRS ${MODULE_LIST})
  set(SUBDIRS ${SUBDIRS} upm)
else()
  subdirlist(SUBDIRS ${CMAKE_CURRENT_SOURCE_DIR})
endif()
foreach(subdir ${SUBDIRS})
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${subdir}/CMakeLists.txt)
        add_subdirectory(${subdir})
    endif()
endforeach()
