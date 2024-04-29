cmake_minimum_required(VERSION 3.17)
#set(CMAKE_TOOLCHAIN_FILE C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake)
project(BGAL)


# Default Build_type RELEASE
set(CMAKE_BUILD_TYPE RELEASE)
# CXX STANDARD
set(CMAKE_CXX_STANDARD 14)
# Set Version
set(VERSION 1.0)
# Set lib output directory
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${PROJECT_SOURCE_DIR}/lib)
#set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

option(CMAKE_INSTALL_PREFIX "BGAL install PATH" /usr/local)
# include header from project_source_dir
include_directories(${PROJECT_SOURCE_DIR}/include)


if(APPLE)
    # openmp support based on: https://stackoverflow.com/a/60198415
    #set(CMAKE_C_COMPILER clang)
    #set(CMAKE_CXX_COMPILER clang++)
    set(OpenMP_ROOT "/opt/homebrew/opt/libomp/") # HACK
        #[[=
    if(CMAKE_C_COMPILER_ID MATCHES "Clang\$")
        set(OpenMP_C_FLAGS "-Xpreprocessor -fopenmp")
        set(OpenMP_C_LIB_NAMES "omp")
        set(OpenMP_omp_LIBRARY omp)
        set(OpenMP_C_INCLUDE_DIR "/opt/homebrew/opt/libomp/include/") # HACK
        set(OpenMP_C_INCLUDE_DIR "/opt/homebrew/opt/libomp/lib/") # HACK
    endif()

    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang\$")
        set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp")
        set(OpenMP_CXX_LIB_NAMES "omp")
        set(OpenMP_omp_LIBRARY omp)
        set(OpenMP_CXX_INCLUDE_DIR "/opt/homebrew/opt/libomp/include/") # HACK
        set(OpenMP_CXX_INCLUDE_DIR "/opt/homebrew/opt/libomp/lib/") # HACK
        add_library_dir(
        set(OpenMP_CXX_LIBRARY "/opt/homebrew/Cellar/libomp/18.1.4/lib/libomp.a") # HACK
    endif()
        #=]]

endif()

include(FetchContent)

set(FETCHCONTENT_QUIET OFF)
set(EXTERNAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external")
set(FETCHCONTENT_UPDATES_DISCONNECTED TRUE)

# Get Boost
find_package(Boost REQUIRED)
if (Boost_FOUND)
    message(STATUS "BOOST FOUNDED")
    include_directories(${Boost_INCLUDE_DIRS})
endif ()

# Get CGAL
find_package(CGAL REQUIRED)
if (CGAL_FOUND)
    include(${CGAL_USE_FILE})
else ()
    message("ERROR: this program requires CGAL and will not be compiled.")
endif ()


if(NOT TARGET Eigen3::Eigen)
    FetchContent_Declare(eigen
        GIT_REPOSITORY https://gitlab.com/libeigen/eigen
        GIT_TAG 3.4.0
        SOURCE_DIR "${EXTERNAL_DIR}/eigen"
    )
    FetchContent_Populate(eigen)
    message("Downloaded Eigen3 to ${eigen_SOURCE_DIR}")
    add_library(Eigen3::Eigen INTERFACE IMPORTED)
    target_include_directories(Eigen3::Eigen INTERFACE "$<BUILD_INTERFACE:${eigen_SOURCE_DIR}>")
    #target_compile_definitions(Eigen3::Eigen INTERFACE -DEIGEN_HAS_STD_RESULT_OF=0)
endif()

link_libraries(Eigen3::Eigen)

find_package(Threads REQUIRED)

if (NOT TARGET libigl::libigl)
    FetchContent_Declare(libigl
        GIT_REPOSITORY https://github.com/libigl/libigl.git
        GIT_TAG dafd52343b1ed5d71450423112404028d38334cb # 2024-04-15
        SOURCE_DIR "${EXTERNAL_DIR}/libigl"
    )
    FetchContent_Populate(libigl)

    add_library(libigl INTERFACE IMPORTED) # "libs/libigl/include/igl/principal_curvature.cpp")
    target_include_directories(libigl INTERFACE "$<BUILD_INTERFACE:${libigl_SOURCE_DIR}/include>")
    target_link_libraries(libigl INTERFACE Eigen3::Eigen)
    target_link_libraries(libigl INTERFACE Threads::Threads)
    add_library(libigl::libigl ALIAS libigl)
endif()


if(WIN32 AND NOT MINGW)
  if(NOT DEFINED CMAKE_DEBUG_POSTFIX)
    set(CMAKE_DEBUG_POSTFIX "d")
  endif()
  if(NOT DEFINED CMAKE_RELWITHDEBINFO_POSTFIX)
    set(CMAKE_RELWITHDEBINFO_POSTFIX "rd")
  endif()
  if(NOT DEFINED CMAKE_MINSIZEREL_POSTFIX)
    set(CMAKE_MINSIZEREL_POSTFIX "s")
  endif()
endif()

# sub_directory
add_subdirectory(include)
add_subdirectory(src)
#target_link_libraries(${PROJECT_NAME}  Algorithm BaseShape Draw Geodesic Integral Model Optimization PointCloudProcessing PQP Reconstruction Tessellation2D Tessellation3D ${Boost_LIBRARIES})
add_subdirectory(MAIN)



install(TARGETS Algorithm BaseShape Draw Geodesic Integral Model Optimization PointCloudProcessing PQP Reconstruction Tessellation2D Tessellation3D CVTLike
	EXPORT ${PROJECT_NAME}Targets
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
	RUNTIME DESTINATION bin
	PUBLIC_HEADER DESTINATION include
	)	
install(DIRECTORY ./include/ DESTINATION include
        FILES_MATCHING PATTERN "*.h")
configure_file(${PROJECT_NAME}-config.cmake.in ${PROJECT_NAME}-config.cmake @ONLY)
install(EXPORT ${PROJECT_NAME}Targets
	FILE ${PROJECT_NAME}-config.cmake
	NAMESPACE ${PROJECT_NAME}::
	DESTINATION lib/cmake/${PROJECT_NAME}
	)
