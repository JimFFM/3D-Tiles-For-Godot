# =============================================================================
# Unified build for the Godot 3D Tiles GDExtension (Windows / MSVC)
#
#   godot-cpp  +  cesium-native  +  the extension  ->  ONE Visual Studio solution
#
# Replaces: CesiumBuildUtils.py (native configure/build), SConstruct + SCsub
# (extension build), and the cesium-native / sdk steps of the old batch script.
# Linux and macOS keep using SCons.
#
# Usage (see build_windows_unified.bat):
#   cmake -S . -B build -G "Visual Studio 18 2026" -A x64
#   cmake --build build --config Release --target Godot3DTiles
# =============================================================================
cmake_minimum_required(VERSION 3.20)

# Only Release is meaningful: the SCons build used template_release, /MT and
# Release cesium-native libraries.
set(CMAKE_CONFIGURATION_TYPES "Release" CACHE STRING "" FORCE)

project(Godot3DTiles LANGUAGES C CXX)

if(NOT WIN32)
  message(FATAL_ERROR "This CMake project covers the Windows build only. "
                      "Use SCons on Linux/macOS.")
endif()

get_property(_is_multi_config GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT _is_multi_config AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
endif()

# -----------------------------------------------------------------------------
# Options
# -----------------------------------------------------------------------------
set(CESIUM_GD_TRIPLET "x64-windows-static" CACHE STRING "vcpkg triplet used for all dependencies")
option(CESIUM_GD_PATCH_EZVCPKG_PIN
       "Replace cesium-native's ezvcpkg commit pin with 2025.09.17 (CI workaround)" ON)
option(CESIUM_GD_INSTALL_EXTRA_VCPKG
       "vcpkg-install uriparser, ada-url and curl into cesium-native's ezvcpkg tree" ON)

option(CESIUM_GD_DOUBLE_PRECISION
       "Build for a double-precision Godot (precision=double in the SCons build)" OFF)

# Double precision needs godot-cpp built with matching precision; only the SCons
# path passes that through reliably, so it is the default there.
if(EXISTS "${CMAKE_SOURCE_DIR}/godot-cpp/CMakeLists.txt" AND NOT CESIUM_GD_DOUBLE_PRECISION)
  set(_gdcpp_default "CMAKE")
else()
  set(_gdcpp_default "SCONS")
endif()
set(CESIUM_GD_GODOT_CPP_BACKEND "${_gdcpp_default}" CACHE STRING
    "How godot-cpp is built: CMAKE (add_subdirectory, appears in the .sln) or SCONS (custom target)")
set_property(CACHE CESIUM_GD_GODOT_CPP_BACKEND PROPERTY STRINGS CMAKE SCONS)

if(CESIUM_GD_DOUBLE_PRECISION AND CESIUM_GD_GODOT_CPP_BACKEND STREQUAL "CMAKE")
  message(FATAL_ERROR "Double precision is only supported with -DCESIUM_GD_GODOT_CPP_BACKEND=SCONS. "
                      "I could not verify how godot-cpp's CMake at this tag selects precision, and a "
                      "precision mismatch between godot-cpp and the extension breaks at runtime.")
endif()

# -----------------------------------------------------------------------------
# Paths
# -----------------------------------------------------------------------------
set(CESIUM_GD_DIR     "${CMAKE_SOURCE_DIR}/cesium_godot")
set(CESIUM_NATIVE_DIR "${CESIUM_GD_DIR}/native")
set(CESIUM_THIRD_PARTY_DIR "${CESIUM_GD_DIR}/third_party")
set(GODOT_CPP_DIR     "${CMAKE_SOURCE_DIR}/godot-cpp")

if(NOT EXISTS "${CESIUM_NATIVE_DIR}/CMakeLists.txt")
  message(FATAL_ERROR "cesium-native not found at ${CESIUM_NATIVE_DIR}. Run build_windows_unified.bat first.")
endif()
if(NOT EXISTS "${GODOT_CPP_DIR}")
  message(FATAL_ERROR "godot-cpp not found at ${GODOT_CPP_DIR}. Run build_windows_unified.bat first.")
endif()

# -----------------------------------------------------------------------------
# One static MSVC runtime (/MT) for every target in this solution.
# Set before any add_subdirectory() so it is inherited by godot-cpp and
# cesium-native.
# -----------------------------------------------------------------------------
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")

# -----------------------------------------------------------------------------
# Source patches (idempotent; they replace the old Python / batch hacks)
# -----------------------------------------------------------------------------

# 1) Material.h: undef OPAQUE (was clean_cesium_definitions() in CesiumBuildUtils.py)
set(_material_h "${CESIUM_NATIVE_DIR}/CesiumGltf/generated/include/CesiumGltf/Material.h")
if(EXISTS "${_material_h}")
  file(READ "${_material_h}" _txt)
  string(FIND "${_txt}" "#undef OPAQUE" _pos)
  if(_pos EQUAL -1)
    string(REPLACE "#pragma once" "#pragma once\n#undef OPAQUE" _txt "${_txt}")
    file(WRITE "${_material_h}" "${_txt}")
    message(STATUS "Patched Material.h (#undef OPAQUE)")
  endif()
else()
  message(WARNING "Material.h not found at ${_material_h}; skipping OPAQUE patch")
endif()

# 2) ezvcpkg commit pin workaround (was an inline python -c in the batch script)
if(CESIUM_GD_PATCH_EZVCPKG_PIN)
  set(_native_cmake "${CESIUM_NATIVE_DIR}/CMakeLists.txt")
  set(_pin_old "COMMIT dbe35ceb30c688bf72e952ab23778e009a578f18")
  set(_pin_new "COMMIT 2025.09.17")
  file(READ "${_native_cmake}" _txt)
  string(FIND "${_txt}" "${_pin_old}" _pos)
  if(NOT _pos EQUAL -1)
    string(REPLACE "${_pin_old}" "${_pin_new}" _txt "${_txt}")
    file(WRITE "${_native_cmake}" "${_txt}")
    message(STATUS "Patched cesium-native ezvcpkg pin -> 2025.09.17")
  else()
    string(FIND "${_txt}" "${_pin_new}" _pos_new)
    if(_pos_new EQUAL -1)
      message(FATAL_ERROR "Expected ezvcpkg commit pin not found in ${_native_cmake}")
    endif()
  endif()
endif()

# -----------------------------------------------------------------------------
# cesium-native (in the same solution)
# Same settings CesiumBuildUtils.configure_native() passed on the command line.
# -----------------------------------------------------------------------------
set(CESIUM_MSVC_STATIC_RUNTIME_ENABLED ON CACHE BOOL "" FORCE)
set(CESIUM_TESTS_ENABLED OFF CACHE BOOL "" FORCE)
set(VCPKG_TRIPLET "${CESIUM_GD_TRIPLET}" CACHE STRING "" FORCE)
set(CMAKE_POLICY_VERSION_MINIMUM 3.5 CACHE STRING "" FORCE)
set(ENV{VCPKG_TRIPLET} "${CESIUM_GD_TRIPLET}")
set(ENV{GIT_LFS_SKIP_SMUDGE} "1")
# Same environment the working CI workflow sets
set(ENV{VCPKG_DEFAULT_TRIPLET} "${CESIUM_GD_TRIPLET}")
set(ENV{VCPKG_FORCE_SYSTEM_BINARIES} "1")
set(ENV{CMAKE_POLICY_VERSION_MINIMUM} "3.5")

# EXCLUDE_FROM_ALL: only the cesium libraries the extension links get built.
add_subdirectory("${CESIUM_NATIVE_DIR}" "${CMAKE_BINARY_DIR}/cesium-native" EXCLUDE_FROM_ALL)

set(CESIUM_GD_CESIUM_TARGETS
  Cesium3DTilesSelection CesiumRasterOverlays Cesium3DTilesContent
  Cesium3DTilesReader CesiumGltfReader CesiumGltf CesiumAsync
  CesiumJsonReader CesiumJsonWriter CesiumUtility CesiumGeospatial
  CesiumGeometry CesiumGltfContent CesiumQuantizedMeshTerrain)
foreach(_t IN LISTS CESIUM_GD_CESIUM_TARGETS)
  if(NOT TARGET ${_t})
    message(FATAL_ERROR "cesium-native target '${_t}' does not exist in this cesium-native version")
  endif()
endforeach()

# -----------------------------------------------------------------------------
# Locate cesium-native's ezvcpkg tree (mirrors find_ezvcpkg_path() in the
# Python utils: <drive root>/.ezvcpkg first, then the home directory; newest
# subfolder wins). Override with -DEZVCPKG_DIR=<path to the versioned folder>.
# -----------------------------------------------------------------------------
function(cesium_gd_find_ezvcpkg out_var)
  if(EZVCPKG_DIR)
    set(${out_var} "${EZVCPKG_DIR}" PARENT_SCOPE)
    return()
  endif()
  string(SUBSTRING "${CMAKE_SOURCE_DIR}" 0 2 _drive)
  set(_roots "${_drive}/.ezvcpkg")
  if(DEFINED ENV{USERPROFILE})
    list(APPEND _roots "$ENV{USERPROFILE}/.ezvcpkg")
  endif()
  if(DEFINED ENV{HOME})
    list(APPEND _roots "$ENV{HOME}/.ezvcpkg")
  endif()

  set(_best "")
  set(_best_time "")
  foreach(_root IN LISTS _roots)
    if(NOT IS_DIRECTORY "${_root}")
      continue()
    endif()
    file(GLOB _subs LIST_DIRECTORIES true "${_root}/*")
    foreach(_d IN LISTS _subs)
      if(IS_DIRECTORY "${_d}/installed/${CESIUM_GD_TRIPLET}/lib")
        file(TIMESTAMP "${_d}" _ts "%Y%m%d%H%M%S" UTC)
        if(_best STREQUAL "" OR _ts STRGREATER _best_time)
          set(_best "${_d}")
          set(_best_time "${_ts}")
        endif()
      endif()
    endforeach()
  endforeach()
  set(${out_var} "${_best}" PARENT_SCOPE)
endfunction()

cesium_gd_find_ezvcpkg(EZVCPKG_PATH)
if(EZVCPKG_PATH STREQUAL "")
  message(FATAL_ERROR "Could not find an ezvcpkg tree with installed/${CESIUM_GD_TRIPLET}. "
                      "cesium-native's configure normally creates it; pass -DEZVCPKG_DIR=... to override.")
endif()
message(STATUS "Using ezvcpkg at ${EZVCPKG_PATH}")

set(EZVCPKG_INSTALLED "${EZVCPKG_PATH}/installed/${CESIUM_GD_TRIPLET}")
set(EZVCPKG_INCLUDE   "${EZVCPKG_INSTALLED}/include")
set(EZVCPKG_LIB       "${EZVCPKG_INSTALLED}/lib")

# Extra vcpkg packages (was install_additional_libs() in the Python utils)
if(CESIUM_GD_INSTALL_EXTRA_VCPKG)
  set(_vcpkg_exe "${EZVCPKG_PATH}/vcpkg.exe")
  if(EXISTS "${_vcpkg_exe}")
    foreach(_spec "uriparser|uriparser.lib" "ada-url|ada.lib" "curl|libcurl.lib")
      string(REPLACE "|" ";" _parts "${_spec}")
      list(GET _parts 0 _pkg)
      list(GET _parts 1 _marker)
      if(NOT EXISTS "${EZVCPKG_LIB}/${_marker}")
        message(STATUS "vcpkg install ${_pkg}:${CESIUM_GD_TRIPLET}")
        # VCPKG_ROOT is pinned to the ezvcpkg tree for this call only. CI sets
        # VCPKG_ROOT to a different folder, and vcpkg.exe can pick that up, which
        # would install into the wrong tree (the SCsub reads the ezvcpkg one).
        execute_process(COMMAND "${CMAKE_COMMAND}" -E env "VCPKG_ROOT=${EZVCPKG_PATH}"
                                "${_vcpkg_exe}" install "${_pkg}:${CESIUM_GD_TRIPLET}"
                        WORKING_DIRECTORY "${EZVCPKG_PATH}"
                        RESULT_VARIABLE _rc)
        if(NOT _rc EQUAL 0)
          message(WARNING "vcpkg install ${_pkg} failed (exit code ${_rc})")
        endif()
      endif()
    endforeach()
  else()
    message(WARNING "vcpkg.exe not found in ${EZVCPKG_PATH}; skipping extra packages")
  endif()
endif()

# -----------------------------------------------------------------------------
# The extension
# -----------------------------------------------------------------------------
set(CESIUM_GD_SOURCES
  cesium_godot/register_types.cpp
  cesium_godot/Models/CesiumGDTileset.cpp
  cesium_godot/Models/GeoreferencedNode.cpp
  cesium_godot/Models/Cesium3DTile.cpp
  cesium_godot/Models/TileMetadata.cpp
  cesium_godot/Models/CesiumGDPanel.cpp
  cesium_godot/Models/CesiumGDRasterOverlay.cpp
  cesium_godot/Models/CesiumGlobe.cpp
  cesium_godot/Models/CesiumGDConfig.cpp
  cesium_godot/Models/CesiumHTTPRequestNode.cpp
  cesium_godot/Models/CesiumIonGDGeoJSON.cpp
  cesium_godot/Models/CesiumGDCreditSystem.cpp
  cesium_godot/Models/CesiumGDGlobeAnchor.cpp
  cesium_godot/Models/CesiumPropertyInfo.cpp
  cesium_godot/Implementations/NetworkAssetAccessor.cpp
  cesium_godot/Implementations/DocumentContainer.cpp
  cesium_godot/Implementations/GodotPrepareRenderResources.cpp
  cesium_godot/CesiumGDModelLoader.cpp
  cesium_godot/Utils/CesiumGDTextureLoader.cpp
  cesium_godot/Utils/CesiumDebugUtils.cpp
  cesium_godot/Utils/AssetManipulation.cpp
  cesium_godot/Utils/CesiumGDAssetBuilder.cpp
  cesium_godot/Utils/TokenTroubleShooting.cpp)

file(GLOB CESIUM_GD_AUX_SOURCES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/cesium_auxiliars/*.cpp")

add_library(Godot3DTiles SHARED ${CESIUM_GD_SOURCES} ${CESIUM_GD_AUX_SOURCES})
target_compile_features(Godot3DTiles PRIVATE cxx_std_20)

# Include paths (same set the SCsub built with load_cesium_module())
set(_inc "${CMAKE_SOURCE_DIR}/testSrc" "${CESIUM_GD_DIR}" "${CMAKE_SOURCE_DIR}/cesium_auxiliars")

set(_native_modules
  Cesium3DTiles Cesium3DTilesContent Cesium3DTilesReader Cesium3DTilesSelection
  Cesium3DTilesWriter CesiumAsync CesiumGeometry CesiumGeospatial CesiumGltf
  CesiumGltfContent CesiumGltfReader CesiumGltfWriter CesiumIonClient
  CesiumJsonReader CesiumJsonWriter CesiumQuantizedMeshTerrain
  CesiumRasterOverlays CesiumUtility
  extern/gsl extern/asyncplusplus extern/fmt extern/spdlog extern/glm
  extern/draco extern/modp extern/rapidjson extern/KHR extern/webp
  extern/misc extern/nonstd extern/s2)
foreach(_m IN LISTS _native_modules)
  list(APPEND _inc "${CESIUM_NATIVE_DIR}/${_m}/include"
                   "${CESIUM_NATIVE_DIR}/${_m}/generated/include")
endforeach()
foreach(_m curl litehtml)
  list(APPEND _inc "${CESIUM_THIRD_PARTY_DIR}/${_m}/include"
                   "${CESIUM_THIRD_PARTY_DIR}/${_m}/generated/include")
endforeach()
list(APPEND _inc "${EZVCPKG_INCLUDE}")
target_include_directories(Godot3DTiles PRIVATE ${_inc})

# Defines / flags (from SConstruct, SCsub and get_compile_flags())
target_compile_definitions(Godot3DTiles PRIVATE
  CESIUM_GD_EXT
  _HAS_EXCEPTIONS=1
  SPDLOG_NO_EXCEPTIONS
  LIBASYNC_NO_EXCEPTIONS)
target_compile_options(Godot3DTiles PRIVATE /Zc:__cplusplus /utf-8 /bigobj /EHsc)
if(CESIUM_GD_DOUBLE_PRECISION)
  target_compile_definitions(Godot3DTiles PRIVATE REAL_T_IS_DOUBLE)
  set(_prec_suffix ".double")
  set(_scons_prec "precision=double")
else()
  set(_prec_suffix "")
  set(_scons_prec "")
endif()
target_link_options(Godot3DTiles PRIVATE /IGNORE:4217)

# Libraries ------------------------------------------------------------------
# cesium-native modules come in as CMake targets (build order + transitive deps)
target_link_libraries(Godot3DTiles PRIVATE ${CESIUM_GD_CESIUM_TARGETS})

# Everything else from vcpkg / third_party, as in the SCsub Windows list.
target_link_directories(Godot3DTiles PRIVATE
  "${EZVCPKG_LIB}"
  "${CESIUM_THIRD_PARTY_DIR}/litehtml")

file(GLOB CESIUM_GD_ABSL_LIBS "${EZVCPKG_LIB}/absl_*.lib")
if(NOT CESIUM_GD_ABSL_LIBS)
  message(WARNING "No absl_*.lib found in ${EZVCPKG_LIB}")
endif()

target_link_libraries(Godot3DTiles PRIVATE
  async++ ktx jpeg turbojpeg draco uriparser s2 glm fmt spdlog tinyxml2
  libcurl sqlite3 libwebp libwebpdecoder zstd libssl meshoptimizer ada
  litehtml gumbo
  zlib zlibstatic-ng libmodpbase64
  ${CESIUM_GD_ABSL_LIBS}
  # Windows system libraries required by the static libcurl
  iphlpapi secur32 ws2_32 crypt32 wldap32 bcrypt normaliz advapi32)

# godot-cpp ------------------------------------------------------------------
if(CESIUM_GD_GODOT_CPP_BACKEND STREQUAL "CMAKE")
  add_subdirectory("${GODOT_CPP_DIR}" "${CMAKE_BINARY_DIR}/godot-cpp")
  target_link_libraries(Godot3DTiles PRIVATE godot-cpp)
else()
  # Fallback: build godot-cpp with its own SCons, then link the static lib.
  # This does NOT put godot-cpp in the .sln; it is a custom build step.
  set(_gdcpp_lib "${GODOT_CPP_DIR}/bin/libgodot-cpp.windows.template_release${_prec_suffix}.x86_64.lib")
  add_custom_command(
    OUTPUT "${_gdcpp_lib}"
    COMMAND scons platform=windows arch=x64 target=template_release use_static_cpp=yes ${_scons_prec}
    WORKING_DIRECTORY "${GODOT_CPP_DIR}"
    COMMENT "Building godot-cpp with SCons"
    VERBATIM)
  add_custom_target(godot_cpp_scons DEPENDS "${_gdcpp_lib}")
  add_dependencies(Godot3DTiles godot_cpp_scons)
  target_include_directories(Godot3DTiles PRIVATE
    "${GODOT_CPP_DIR}/include" "${GODOT_CPP_DIR}/gen/include" "${GODOT_CPP_DIR}/gdextension")
  target_compile_definitions(Godot3DTiles PRIVATE TYPED_METHOD_BIND WINDOWS_ENABLED NOMINMAX)
  target_link_libraries(Godot3DTiles PRIVATE "${_gdcpp_lib}")
endif()

# Output: same file name and folder the SCons build produced -----------------
set(_out_dir "${CMAKE_SOURCE_DIR}/godot3dtiles/addons/cesium_godot/lib")
set_target_properties(Godot3DTiles PROPERTIES
  PREFIX ""
  OUTPUT_NAME "Godot3DTiles.windows.template_release${_prec_suffix}.x86_64"
  RUNTIME_OUTPUT_DIRECTORY "${_out_dir}"
  RUNTIME_OUTPUT_DIRECTORY_RELEASE "${_out_dir}")

set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT Godot3DTiles)
