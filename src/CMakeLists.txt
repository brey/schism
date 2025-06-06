
############# CMake versioning and backward compatibility #####
cmake_minimum_required (VERSION 3.12)
set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

# This policy has to do with variables called <module>_ROOT
# and possible collisions with module-finding.
cmake_policy(SET CMP0074 OLD)


enable_testing()

############ Include paths for CMake ####
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/../cmake/modules")
set(CMAKE_USER_MAKE_RULES_OVERRIDE "${CMAKE_SOURCE_DIR}/../cmake/SCHISMCompile.cmake")
include(MPIWrap)
include(MakeDefineOption)
include(SCHISMTest)


message(STATUS "\n################   Begin SCHISM Configuration   ################" )
message(STATUS "In-source builds not allowed")

set(comp_lang_list C Fortran)
project(SCHISM LANGUAGES ${comp_lang_list} CXX)
add_definitions("-DSCHISM")

########### Announce compiler #####
message(STATUS "\n### Compiler combination and executables")
message(STATUS "Compiler ID is ${CMAKE_Fortran_COMPILER_ID}-${CMAKE_C_COMPILER_ID}")
message(STATUS "Fortran compiler is ${CMAKE_Fortran_COMPILER}")
message(STATUS "C compiler is ${CMAKE_C_COMPILER}")

########### Build types are Release, Debug and RelWithDebInfo (Release with debug info) #####
if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "\n### No CMAKE_BUILD_TYPE selected in command or cache, default to Release. \nAlternatives are Debug and RelWithDebInfo")
    set(CMAKE_BUILD_TYPE "Release")
endif()

########### C Preprocessor #####
# The thought is to enable this globally, since it is needed for all
# of SCHISM and several Utilities
# @todo this might be more elegantly handled (from cmake 3.18) with FORTRAN_PREPROCESS boolean flag
if (NOT DEFINED C_PREPROCESS_FLAG)
  message(FATAL_ERROR "Preprocessor flag (e.g. -cpp for intel, -MPreprocess for PGI0 is not set")
endif()
add_compile_options(${C_PREPROCESS_FLAG})

######### Algorithmic switches and build options involving functionality

message(STATUS "\n### Processing algorithmic switches and build options involving functionality for main project")

# Defined early in the list to allow use as a default for others
define_opt(USE_BMI "Build a library that can be used from BMI, with sources and some b.c. provided by BMI" OFF)

define_opt(NO_PARMETIS "Disable ParMETIS" OFF)
define_opt(DEBUG "Enable diagnostics (slow and not suitable for production simulations" OFF)
define_opt(USE_ANALYSIS "Triggers detailed calculations and output options" OFF)
define_opt(INCLUDE_TIMING "Add timing profile instrumentation to simulation" OFF)
define_opt(TIMER2 "Print timing information" OFF)
define_opt(OLDIO "Old nc output option" OFF)
define_opt(USE_ATMOS "Atmospheric model output option" OFF)
define_opt(PREC_EVAP "Use  precipitation/evaporation option" OFF)
define_opt(IMPOSE_NET_FLUX "Specify net heat and salt fluxes in sflux" OFF)
define_opt(USE_BULK_FAIRALL "Enable Fairall scheme" OFF)
define_opt(USE_PAHM "Enable USE_PAHM scheme" OFF)
define_opt(USE_HA "Enable Harmonic Analysis output" OFF)
define_opt(USE_SIMPLE_WIND "Simplified wind data interface" OFF)
define_opt(USE_GOTM  "Enable compiling with GOTM turbulence closure" OFF)
define_opt(USE_MARSH "Enable Marsh module" OFF)
define_opt(DEBUG "Enable debug mode" OFF)
define_opt(USE_GEN "Enable generic tracer module" OFF)
define_opt(USE_AGE "Enable age tracer module" OFF)
define_opt(ICM_PH "Enable pH module in ICM" OFF)
define_opt(USE_DVD "Enable DVD module" OFF)
define_opt(BLD_STANDALONE "Standalone build mode" ON)
define_opt(SH_MEM_COMM "Shared mem communicator" OFF)
define_opt(BUILD_TOOLS "Build pschism and all the utilities" ON)
define_opt(BUILD_SHARED_LIBS "Build SCHISM shared libraries" OFF)

if(NOT DEFINED TVD_LIM)
  message(WARNING "TVD_LIM must be provided, either in the cache. It must be one of SB, VL, MM or OS for Superbee, Van Leer, Minmod, or Osher. Default is VL")
endif()
set(TVD_LIM VL CACHE STRING "Flux limiter. SB, VL, MM or OS for Superbee, Van Leer, Minmod, or Osher")
list(APPEND def_tags _TVD-${TVD_LIM})

string(CONCAT all_def_tags ${def_tags})
string(REPLACE USE_ "" def_tag_rev ${all_def_tags})
string(REPLACE INCLUDE_ "" def_tag_rev ${def_tag_rev})

message(STATUS "Build definition tags set to ${def_tag_rev}")


########### MPI Library ########
# The typical pattern with cmake is to use the mpi wrapper (e.g. mpif90)
# to infer flags and locations for the compiler
message(STATUS "\n### Configuring MPI Library")
if ($ENV{MPI_VERSION})
   set(MPIVERSION_DEFAULT $ENV{MPI_VERSION})
else ($ENV{MPI_VERSION})
   set(MPIVERSION_DEFAULT 2)
endif()
set(MPIVERSION ${MPIVERSION_DEFAULT} CACHE STRING "Version of MPI API (1 or 2)") 

find_package(MPI REQUIRED)
message(STATUS "MPIVERSION = ${MPIVERSION}")
add_compile_definitions(MPIVERSION=${MPIVERSION})

#set_property(GLOBAL APPEND PROPERTY DEFINE-LIST "MPIVERSION=${MPIVERSION}")

add_definitions(${MPI_Fortran_COMPILE_FLAGS})
if (CMAKE_Fortran_FLAGS)
  set(CMAKE_Fortran_FLAGS ${CMAKE_Fortran_FLAGS} ${MPI_Fortran_COMPILE_FLAGS})
endif (CMAKE_Fortran_FLAGS)
include_directories(${MPI_Fortran_INCLUDE_PATH})

add_definitions(${MPI_C_COMPILE_FLAGS})
if (CMAKE_C_FLAGS)
  set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} ${MPI_C_COMPILE_FLAGS})
endif (CMAKE_C_FLAGS)
include_directories(${MPI_C_INCLUDE_PATH})

########### NetCDF Dependency is handled here because so many libraries use it ##############
if (BLD_STANDALONE)
  message(STATUS "\n### Configuring NetCDF ")
  #set(NetCDF_DIR "/usr/local/dms/pkg/netcdf/4.1.3-intel12.0-parallel")
  if( NOT DEFINED NetCDF_DIR AND DEFINED NETCDF_DIR)
    message(WARNING "Note the mixed case of NetCDF_DIR if you put it in your cache. This CMake variable is mixed case, different from the typical environment NETCDF_DIR. Proceeding anyway.")
    set(NetCDF_C_DIR ${NETCDF_C_DIR})
  endif()
  
  if( NOT DEFINED NetCDF_C_DIR AND ($ENV{NETCDF_C_DIR} MATCHES ".+") )
    message(STATUS "Using NETCDF_C_DIR location from environment.")
    set(NetCDF_C_DIR $ENV{NETCDF_C_DIR})
  endif()
  
  if( NOT DEFINED NetCDF_FORTRAN_DIR AND ($ENV{NETCDF_FORTRAN_DIR} MATCHES ".+") )
    message(STATUS "Using NETCDF_FORTRAN_DIR location from environment.")
    set(NetCDF_FORTRAN_DIR $ENV{NETCDF_FORTRAN_DIR})
  endif()
  
  find_package(NetCDF REQUIRED COMPONENTS ${comp_lang_list})
  if(NetCDF_NEEDS_HDF5)
    set( HDF5_USE_STATIC_LIBRARIES 1 )
    find_package(HDF5 COMPONENTS C REQUIRED)
    # link_directories is bad cmake form, and was used because I would have had to add a lot to 2.6 to get the magic
    # these next couple lines can be removed if we uniformly require cmake 2.8
    set( HDF_LIB_NAMES "hdf5_hl;hdf5;z" )
    link_directories( ${HDF5_LIBRARY_DIRS} )
    if(HDF5_FOUND)
      message(STATUS "HDF5 found")
    endif(HDF5_FOUND)
  endif(NetCDF_NEEDS_HDF5)
  include_directories( ${NetCDF_INCLUDE_DIRS} )

  set (NetCDFLIBS ${NetCDF_LIBRARIES} ${HDF_LIB_NAMES})
endif()

# this might help to use old or pre-compiled GOTM versions:
#if (USE_GOTM)
#  if (NOT DEFINED GOTM_DIR)
#    if( DEFINED $ENV{GOTM_DIR} )
#      set( GOTM_DIR ${ENV{GOTM_DIR} CACHE PATH "Path to GOTM")
#    else( DEFINED ${ENV{GOTM_DIR})
#     message(STATUS "GOTM_DIR not in the environment, setting to local copy -- should be pre built")
#     set(GOTM_DIR "${PROJECT_SOURCE_DIR}/GOTM3.2.5" CACHE PATH "Path to GOTM Library")
#    endif( DEFINED ${ENV{GOTM_DIR} )
#  endif(NOT DEFINED GOTM_DIR)
#  find_package(GOTM REQUIRED)
#  set(GOTMLIBS ${GOTM_LIBRARIES} )
#  message("GOTM_INCLUDE_DIRS: ${GOTM_INCLUDE_DIRS}")
#  include_directories(${GOTM_INCLUDE_DIRS})
#endif(USE_GOTM)

########### Put each type of output (exe, libraries, modules) in the normal places
###         This file is untested for compilers with no output module directory option
message(STATUS "\n### Assigning output directories for binaries (bin), modules (include) and libraries (lib)")
if (BLD_STANDALONE)
  set ( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
#  set ( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin
#    CACHE PATH "Output Directory for executables.")

  set ( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
#  set ( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
#    CACHE PATH "Output Directory for all static libraries.")

  set ( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
#  set ( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
#    CACHE PATH "Output Directory for all shared libraries.")
else()
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
endif()

if( CMAKE_Fortran_MODDIR_FLAG)
  SET( CMAKE_Fortran_MODULE_DIRECTORY ${PROJECT_BINARY_DIR}/include
  CACHE PATH "Output location for include files and fortran modules")
  include_directories("${CMAKE_Fortran_MODULE_DIRECTORY}")
endif(CMAKE_Fortran_MODDIR_FLAG)


########### Process libraries ######
message(STATUS "\n### Adding enabled libraries to project") ######

# Macro for adding a library module to avoid boilerplate:
#     name:  directory for module (e.g. EcoSim)
#     option_name: compiler -D definition (e.g. USE_ECO)
#     default: ON/OFF Should generally be off except Hydro

macro( add_module name option_name  default)
  option(${option_name} "Use ${lib_name} module" ${default})
  if (${option_name})
    list(APPEND all_use_mod_defs "${option_name}")
    list(APPEND schismmoddirs ${name})
  endif(${option_name})
endmacro(add_module)




# Create list of modules
add_module(WWMIII    USE_WWM   OFF)
add_module(WW3    USE_WW3   OFF)
add_module(COSINE USE_COSINE OFF)
add_module(ICM      USE_ICM OFF)
add_module(FIB      USE_FIB OFF)
add_module(Ice      USE_ICE   OFF)
add_module(Multi_ice USE_MICE   OFF)
add_module(Fabm     USE_FABM  OFF)
add_module(EcoSim   USE_ECO   OFF)
add_module(Sediment USE_SED   OFF)
add_module(Sed2d    USE_SED2D OFF)
add_module(Hydro    USE_HYDRO ON)
add_module(Core     USE_SCHISM ON)

if(NOT NO_PARMETIS)
  ########## BEG:: PARMETIS ##########
  ### Panagiotis Velissariou 08/25/2022
  ### Code block to allow the use of an externally compiled ParMETIS library.
  ### The user can supply one of the following variables to provide the full path
  ### to the installed ParMETIS librarie(s):
  ### (a) Via the cmake command line:
  ###   -DPARMETIS_HOME=PATH, or -DPARMETISHOME=PATH, or
  ###   -DPARMETIS_ROOT=PATH, or -DPARMETISROOT=PATH, or
  ###   -DPARMETIS_PATH=PATH, or -DPARMETISPATH=PATH
  ### (b) Via the environment:
  ###   PARMETIS_HOME=PATH, or PARMETISHOME=PATH, or
  ###   PARMETIS_ROOT=PATH, or PARMETISROOT=PATH, or
  ###   PARMETIS_PATH=PATH, or PARMETISPATH=PATH
  ###  We keep PARMETIS_DIR for internal use only.
  ### NOTE: If the supplied PATH is not a valid directory, then the PARMETIS_DIR
  ###       below is not set and the compilation proceeds with the SCHISM's
  ###       internal ParMetis library source.
  message(STATUS "\n### Configuring ParMETIS ")
  unset(PARMETIS_DIR CACHE)
  unset(ENV{PARMETIS_DIR})
  foreach( pm_sfx IN ITEMS HOME ROOT PATH)
    foreach( pm_sep IN ITEMS "_" "" )
      set(pm_tmp_nam "PARMETIS${pm_sep}${pm_sfx}")
      if(DEFINED ${pm_tmp_nam})
        if(NOT "${${pm_tmp_nam}}" STREQUAL "")
          get_filename_component(pm_tmp_val "${${pm_tmp_nam}}" ABSOLUTE REALPATH)
          if(IS_DIRECTORY ${pm_tmp_val})
            set(PARMETIS_DIR "${pm_tmp_val}")
            break()
          endif()
        endif()
      elseif(DEFINED ENV{${pm_tmp_nam}})
        if(NOT "$ENV{${pm_tmp_nam}}" STREQUAL "")
          get_filename_component(pm_tmp_val "$ENV{${pm_tmp_nam}}" ABSOLUTE REALPATH)
          if(IS_DIRECTORY ${pm_tmp_val})
            set(PARMETIS_DIR "${pm_tmp_val}")
            break()
          endif()
        endif()
      endif()
    endforeach()
  endforeach()
  unset(pm_sfx)
  unset(pm_sep)
  unset(pm_tmp_nam)
  unset(pm_tmp_val)

  if(DEFINED PARMETIS_DIR)
    message(STATUS "Trying to Use an externally compiled ParMETIS library.")

    find_package(ParMETIS REQUIRED C CXX)

    if(PARMETIS_EXTERNAL_FOUND)
      set(PARMETIS_DIR "${PARMETIS_DIR}/" CACHE PATH "Path to already compiled ParMetis external library." FORCE)
      message(STATUS "Using ParMETIS in: ${PARMETIS_DIR}")
    else()
      message(WARNING "The externaly compiled ParMETIS library could not be found. Reverting back to SCHISM's internal ParMETIS.")
      unset(PARMETIS_DIR CACHE)
    endif()
  endif()

  if(NOT DEFINED PARMETIS_DIR)
    message(STATUS "Using SCHISM's internal ParMETIS library.")

    set(PARMETIS_VER ParMetis-4.0.3 CACHE STRING "Version and relative path to Parmetis library inside /src.")
    set(PARMETIS_DIR "${CMAKE_SOURCE_DIR}/${PARMETIS_VER}/" CACHE PATH "Path to ParMetis which defaults to /src/PARMETIS_VER in the schism source directory. Support for other locations not tested." FORCE)
    add_subdirectory( ${PARMETIS_VER} )

    message(STATUS "PARMETIS_DIR: ${PARMETIS_DIR}")
    message(STATUS "PARMETIS_VER: ${PARMETIS_VER}")
  endif()
  ########## END:: PARMETIS ##########
endif(NOT NO_PARMETIS)

# Now recurse them and process the contents. 
# In the process, we accumulate a (global) list of compile options and libraries
foreach (moddir ${schismmoddirs})
   add_subdirectory( ${moddir} )
   #if(EXISTS "${PROJECT_SOURCE_DIR}/../test/unittest/${moddir}/CMakeLists.txt")
   #  add_subdirectory( "${PROJECT_SOURCE_DIR}/../test/unittest/${moddir}" "${PROJECT_BINARY_DIR}/unittest/${moddir}" )
   # endif()
    message(STATUS "Added ${moddir}")
endforeach()

######## Executable Name

# Create string to append to executable name with module names for the executable name
string(CONCAT mod_tag ${all_use_mod_defs})
string(REPLACE USE_ _ mod_tag_rev ${mod_tag})
string(REPLACE _HYDRO_SCHISM "" mod_tag_rev ${mod_tag_rev})


######## Tests ##################

# Note this is fine for the unit test directory but it may not be what we want because it doesn't pick and 
# choose the subdirectories according to what modules got selected
#add_subdirectory(${PROJECT_SOURCE_DIR}/../test/unittest/${moddir} ${PROJECT_BINARY_DIR}/unittest/${moddir})

######## GOTM ############
# call cmake with -DUSE_GOTM=ON -DGOTM_BASE=[gotm code directory]
#
if(USE_GOTM)
find_path(GOTM_BASE src/gotm/gotm.F90 $GOTM_BASE ${CMAKE_CURRENT_SOURCE_DIR}/GOTM5.2/code DOC "GOTM source directory")
if(GOTM_BASE)
  message(STATUS "using GOTM_BASE ${GOTM_BASE}")
  set(GOTM_BUILD_LIBRARIES_ONLY ON)
  set(GOTM_USE_FABM OFF CACHE BOOL "use GOTM without FABM")
  set(GOTM_USE_FLEXIBLE_OUTPUT OFF CACHE BOOL "use GOTM without flexible output")
  add_subdirectory(${GOTM_BASE}/src gotm)
  set(GOTM_INCLUDE_DIRS ${CMAKE_BINARY_DIR}/gotm/modules)
  set(GOTM_LIBRARIES turbulence util)
  add_dependencies(hydro ${GOTM_LIBRARIES})
  set(GOTMLIBS ${GOTM_LIBRARIES} )
  set_property(TARGET hydro APPEND PROPERTY INCLUDE_DIRECTORIES "${GOTM_INCLUDE_DIRS}")
endif(GOTM_BASE)
endif(USE_GOTM)

########  Main executable #######
get_property(all_define_options GLOBAL PROPERTY DEFINE-LIST)
message(STATUS "All defines: ${all_define_options}")
# Now that the list of global options are accumulated, apply it on all the directories
foreach (moddir ${schismmoddirs})
set_property( DIRECTORY ${moddir} APPEND PROPERTY COMPILE_DEFINITIONS ${all_define_options} ${all_use_mod_defs})
endforeach()

# Set the driver information for BLD_STANDALONE
if (BLD_STANDALONE OR BUILD_TOOLS)
  message(STATUS "\n### Configuring Driver")
  add_subdirectory( Driver )
  set_property( DIRECTORY "Driver" APPEND PROPERTY COMPILE_DEFINITIONS ${all_define_options} ${all_use_mod_defs})
endif()

######## BMI Implementation Module #######
if (USE_BMI)
  add_subdirectory(BMI)
endif(USE_BMI)

######## Utilities ########
if (BLD_STANDALONE OR BUILD_TOOLS)
message(STATUS "\n### Configuring Utilities")
add_subdirectory( Utility )
endif()
