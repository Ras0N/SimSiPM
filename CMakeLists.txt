cmake_minimum_required(VERSION 3.12)
project(SimSiPM VERSION 1 DESCRIPTION "Library for SiPM simulation" LANGUAGES CXX)
if(MSVC)
	set(CMAKE_CXX_FLAGS_INIT /fp:fast)
	include(FeatureSummary)
else()
	set(CMAKE_CXX_FLAGS_INIT -ffast-math)
	include(GNUInstallDirs)
	include(CMakePackageConfigHelpers)
	include(CMakeDependentOption)
	include(FeatureSummary)
endif()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING
    "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
    FORCE)
endif(NOT CMAKE_BUILD_TYPE)

set(SIPM_BUILD_PYTHON OFF CACHE BOOL "Compile python bindings for SiPM simulation library")
set(SIPM_ENABLE_TEST OFF CACHE BOOL "Build tests for SiPM simulation library")
set(SIPM_ENABLE_BENCH OFF CACHE BOOL "Build benchmarks for SiPM simulation library")
set(SIPM_BUILD_DOCS OFF CACHE BOOL "Build documentation for SiPM simulation library")
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# set(SIPM_BUILD_PYTHON ON)

if (CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
	if(SIPM_ENABLE_TEST)
	    add_subdirectory(tests)
	endif(SIPM_ENABLE_TEST)
endif ()

if (CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  if(SIPM_ENABLE_BENCH)
	    add_subdirectory(bench)
    endif(SIPM_ENABLE_BENCH)
endif ()

if(SIPM_BUILD_DOCS)
  add_subdirectory(docs)
endif()


# Get files
file(GLOB_RECURSE src "src/*.cpp")
file(GLOB_RECURSE include "include/*.h")
file(GLOB_RECURSE pysrc "python/*.cpp")

# Library
add_library(sipm SHARED
  ${src}
  ${include}
)

# Include files
target_include_directories(sipm PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

set_target_properties(sipm PROPERTIES VERSION 1 OUTPUT_NAME sipm)
set_property(TARGET sipm PROPERTY PUBLIC_HEADER ${include})

feature_summary(WHAT ALL)

# Install
install(TARGETS sipm EXPORT sipm-config
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
	PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/sipm
)

if(MSVC AND CMAKE_BUILD_TYPE STREQUAL "Debug")
	install(FILES $<TARGET_PDB_FILE:sipm>
		DESTINATION ${CMAKE_INSTALL_BINDIR}/Debug
		OPTIONAL
	)
endif()

install(EXPORT sipm-config
	FILE sipm-config.cmake
	DESTINATION "${CMAKE_INSTALL_DATADIR}/sipm/cmake"
	NAMESPACE sipm::
)

if (SIPM_BUILD_PYTHON)
	set(PYBIND11_FINDPYTHON ON)
	# In windows, we should find pybind11 install directory ourself.
	find_package(Python COMPONENTS Interpreter Development)
	message(STATUS "Using python executable at ${Python_EXECUTABLE}")
	message(STATUS "installed in ${Python_SITEARCH}")
	if(WIN32)
		execute_process(
			COMMAND ${Python_EXECUTABLE} -c "import pybind11;print(pybind11.get_cmake_dir())"
			OUTPUT_VARIABLE pybind11_DIR
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)
		if (NOT EXISTS "${pybind11_DIR}/pybind11-config.cmake")
			if(NOT EXISTS "${pybind11_DIR}/pybind11config.cmake")
				set(pybind11_DIR "${Python_SITEARCH}/pybind11/share/cmake/pybind11")
			endif()
		endif()
		find_package(pybind11 CONFIG REQUIRED HINTS ${pybind11_DIR})
	else()
		find_package(pybind11 CONFIG REQUIRED)
	endif()
	pybind11_add_module(SiPM MODULE ${pysrc} ${src} ${include})
	target_include_directories(SiPM PRIVATE 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	)
	set_property(TARGET SiPM PROPERTY CXX_STANDARD 17)
	if(MSVC)
		target_compile_options(
			SiPM 
			PRIVATE 
			/W4 /MP
			$<$<CONFIG:Debug>:/fp:fast /Od /Zi>
			$<$<CONFIG:Release>:/fp:fast /O2 /DNDEBUG>
		)
	else()
		target_compile_options(SiPM PRIVATE -fvisibility=hidden -ffast-math -O3)
	endif()
	if(WIN32)
		file(TO_CMAKE_PATH ${Python_SITEARCH} install_path)
		install(TARGETS SiPM LIBRARY DESTINATION ${install_path})
	else()
		install(TARGETS SiPM
		LIBRARY DESTINATION lib/python${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}/site-packages
		)
	endif()
endif (SIPM_BUILD_PYTHON)
