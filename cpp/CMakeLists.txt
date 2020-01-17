cmake_minimum_required(VERSION 3.11)
project(miniWeather CXX CUDA)

enable_testing()


############################################################
## Set Parameters
############################################################
if ("${NX}" STREQUAL "")
  SET(NX 100)
endif()
if ("${NZ}" STREQUAL "")
  SET(NZ 50)
endif()
if ("${SIM_TIME}" STREQUAL "")
  SET(SIM_TIME 1000)
endif()
if ("${OUT_FREQ}" STREQUAL "")
  SET(OUT_FREQ 10)
endif()
if ("${DATA_SPEC}" STREQUAL "")
  SET(DATA_SPEC DATA_SPEC_THERMAL)
endif()
SET(EXE_DEFS "-D_NX=${NX} -D_NZ=${NZ} -D_SIM_TIME=${SIM_TIME} -D_OUT_FREQ=${OUT_FREQ} -D_DATA_SPEC=${DATA_SPEC}")
SET(TEST_DEFS "-D_NX=100 -D_NZ=50 -D_SIM_TIME=400 -D_OUT_FREQ=400 -D_DATA_SPEC=DATA_SPEC_THERMAL")


############################################################
## Check for Parallel NetCDF
############################################################
if ("${PNETCDF_PATH}" STREQUAL "")
  MESSAGE(FATAL_ERROR "ERROR: PNETCDF_PATH CMake variable not set.\n"
                      "miniWeather requires a parallel-netcdf installation <https://trac.mcs.anl.gov/projects/parallel-netcdf>.\n"
                      "Specify -DPNETCDF_PATH=/path/to/pnetcdf in your cmake configure line.\n\n")
endif()


############################################################
## Append CXXFLAGS
############################################################
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXXFLAGS}")


############################################################
## Add YAKL 
############################################################
add_subdirectory(YAKL)
include_directories(YAKL)
include_directories(cub)
include_directories(hipCUB)
include_directories(rocPRIM)


############################################################
## Add Kokkos 
############################################################
if ("${ARCH}" STREQUAL "CUDA")
#  set(KOKKOS_ENABLE_CUDA True)
#  set(KOKKOS_ENABLE_CUDA_LAMBDA True)
endif()
add_subdirectory(kokkos)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/kokkos)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/kokkos/core/src)


############################################################
## Compile the serial version
############################################################
add_executable(serial miniWeather_serial.cpp)
target_include_directories(serial PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(serial PROPERTIES COMPILE_FLAGS "${EXE_DEFS}")
set(_LIBS "-L${PNETCDF_PATH}/lib -lpnetcdf ${LDFLAGS}")
string(STRIP "${_LIBS}" _LIBS)
target_link_libraries(serial kokkos "${_LIBS}")

add_executable(serial_test miniWeather_serial.cpp)
target_include_directories(serial_test PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(serial_test PROPERTIES COMPILE_FLAGS "${TEST_DEFS}")
target_link_libraries(serial_test kokkos "${_LIBS}")

add_test(NAME SERIAL_TEST COMMAND ./check_output.sh ./serial_test 1e-9 4.5e-5 )


############################################################
## Compile the MPI version
############################################################
add_executable(mpi miniWeather_mpi.cpp)
target_include_directories(mpi PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(mpi PROPERTIES COMPILE_FLAGS "${EXE_DEFS}")
set(_LIBS "-L${PNETCDF_PATH}/lib -lpnetcdf ${LDFLAGS}")
string(STRIP "${_LIBS}" _LIBS)
target_link_libraries(mpi kokkos "${_LIBS}")

add_executable(mpi_test miniWeather_mpi.cpp)
target_include_directories(mpi_test PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(mpi_test PROPERTIES COMPILE_FLAGS "${TEST_DEFS}")
target_link_libraries(mpi_test kokkos "${_LIBS}")

add_test(NAME MPI_TEST COMMAND ./check_output.sh ./mpi_test 1e-9 4.5e-5 ) 


############################################################
## YAKL Parallel For Version
############################################################
if ("${ARCH}" STREQUAL "CUDA")
  message(STATUS "Compiling YAKL code for CUDA")
  set(CUDA_DEFS "-D__USE_CUDA__")
  set_source_files_properties(miniWeather_mpi_yakl.cpp PROPERTIES LANGUAGE CUDA)
  set_source_files_properties(miniWeather_mpi_yakl.cpp PROPERTIES COMPILE_FLAGS "${CUDA_FLAGS} -std=c++11 -ccbin=mpic++ --expt-extended-lambda")
elseif("${ARCH}" STREQUAL "HIP")
  SET(EXE_DEFS "${EXE_DEFS} -D__USE_HIP__")
endif()

add_executable(yaklexe miniWeather_mpi_yakl.cpp)
target_include_directories(yaklexe PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(yaklexe PROPERTIES COMPILE_FLAGS "${CUDA_DEFS} ${EXE_DEFS} ${OPENMP_FLAGS}" LINKER_LANGUAGE CUDA)
set(_LIBS "-L${PNETCDF_PATH}/lib -lpnetcdf ${LDFLAGS} ${OPENMP_FLAGS}")
string(STRIP "${_LIBS}" _LIBS)
target_link_libraries(yaklexe yakl "${_LIBS}")

add_executable(yaklexe_test miniWeather_mpi_yakl.cpp)
target_include_directories(yaklexe_test PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(yaklexe_test PROPERTIES COMPILE_FLAGS "${CUDA_DEFS} ${TEST_DEFS} ${OPENMP_FLAGS}" LINKER_LANGUAGE CUDA)
target_link_libraries(yaklexe_test yakl "${_LIBS}")

add_test(NAME YAKL_TEST COMMAND ./check_output.sh ./yaklexe_test 1e-9 4.5e-5 )


############################################################
## KOKKOS Parallel For Version
############################################################
set(CUDA_DEFS )
#if ("${ARCH}" STREQUAL "CUDA")
#  message(STATUS "Compiling Kokkos code for CUDA")
#  set(CUDA_DEFS "-D__USE_CUDA__")
#  set_source_files_properties(miniWeather_mpi_kokkos.cpp PROPERTIES LANGUAGE CUDA)
#  set_source_files_properties(miniWeather_mpi_kokkos.cpp PROPERTIES COMPILE_FLAGS "-std=c++11 -ccbin=mpic++ --expt-extended-lambda")
#elseif("${ARCH}" STREQUAL "HIP")
#  message(STATUS "WARNING: Kokkos has no HIP backend yet")
#endif()

add_executable(kokkosexe miniWeather_mpi_kokkos.cpp)
target_include_directories(kokkosexe PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(kokkosexe PROPERTIES COMPILE_FLAGS "${CUDA_DEFS} ${EXE_DEFS} ${OPENMP_FLAGS}" LINKER_LANGUAGE CUDA)
set(_LIBS "-L${PNETCDF_PATH}/lib -lpnetcdf ${LDFLAGS} ${OPENMP_FLAGS}")
string(STRIP "${_LIBS}" _LIBS)
target_link_libraries(kokkosexe kokkos "${_LIBS}")

add_executable(kokkosexe_test miniWeather_mpi_kokkos.cpp)
target_include_directories(kokkosexe_test PUBLIC ${PNETCDF_PATH}/include)
set_target_properties(kokkosexe_test PROPERTIES COMPILE_FLAGS "${CUDA_DEFS} ${TEST_DEFS} ${OPENMP_FLAGS}" LINKER_LANGUAGE CUDA)
target_link_libraries(kokkosexe_test kokkos "${_LIBS}")

add_test(NAME KOKKOS_TEST COMMAND ./check_output.sh ./kokkosexe_test 1e-9 4.5e-5 )


