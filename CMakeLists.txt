# Copyright (c) 2022 Ultimaker B.V.
# CuraEngine is released under the terms of the AGPLv3 or higher.

cmake_policy(SET CMP0091 NEW)  # For MSVC flags, will be ignored on non-Windows OS's
cmake_minimum_required(VERSION 3.20)
project(CuraEngine LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Replace standardprojectsettings (Conan dependency) with minimal implementation
function(AssureOutOfSourceBuilds)
    if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
        message(FATAL_ERROR "In-source builds are not allowed. Please create a separate build directory and run cmake from there.")
    endif()
endfunction()
AssureOutOfSourceBuilds()

# Minimal replacements for standardprojectsettings functions
function(enable_sanitizers target)
    # Disable sanitizers for release build - optional for local builds
endfunction()

function(set_project_warnings target)
    # Optional: can add warnings here if desired
endfunction()

function(use_threads target)
    find_package(Threads REQUIRED)
    target_link_libraries(${target} PUBLIC Threads::Threads)
endfunction()

option(ENABLE_ARCUS "Enable support for ARCUS" ON)
option(ENABLE_TESTING "Build with unit tests" OFF)
option(ENABLE_BENCHMARKS "Build with Benchmarks" OFF)
option(EXTENSIVE_WARNINGS "Build with all warnings" ON)
option(ENABLE_PLUGINS "Build with plugins" ON)
option(ENABLE_REMOTE_PLUGINS "Build with all warnings" OFF)
option(ENABLE_SENTRY "Send crash data via Sentry" OFF)
option(ENABLE_MORE_COMPILER_OPTIMIZATION_FLAGS "Enable more optimization flags" ON)
option(USE_SYSTEM_LIBS "Use the system libraries if available" OFF)
option(OLDER_APPLE_CLANG "Apple Clang <= 13 used" OFF)
option(ENABLE_THREADING "Enable threading support" ON)

if (${ENABLE_ARCUS} OR ${ENABLE_PLUGINS})
    find_package(protobuf REQUIRED)
endif ()

MESSAGE(STATUS "Building with plugins support: ${ENABLE_PLUGINS}")
if (${ENABLE_PLUGINS})
    find_package(asio-grpc REQUIRED)
    find_package(gRPC REQUIRED)
    find_package(curaengine_grpc_definitions REQUIRED)
    find_package(semver REQUIRED)
    MESSAGE(STATUS "Plugin secure remotes allowed: ${ENABLE_REMOTE_PLUGINS}")
endif ()

message(STATUS "Building with Arcus: ${ENABLE_ARCUS}")
if (${ENABLE_ARCUS})
    find_package(arcus REQUIRED)
    if(TARGET Arcus::Arcus AND NOT TARGET arcus::arcus)
        add_library(arcus::arcus ALIAS Arcus::Arcus)
    endif()
    protobuf_generate_cpp(engine_PB_SRCS engine_PB_HEADERS Cura.proto)
endif ()

### Compiling CuraEngine ###
# First compile all of CuraEngine as library.

set(engine_SRCS # Except main.cpp.
        src/Application.cpp
        src/ConicalOverhang.cpp
        src/ExtruderPlan.cpp
        src/ExtruderTrain.cpp
        src/FffGcodeWriter.cpp
        src/FffPolygonGenerator.cpp
        src/FffProcessor.cpp
        src/gcodeExport.cpp
        src/GCodePathConfig.cpp
        src/infill.cpp
        src/InterlockingGenerator.cpp
        src/InsetOrderOptimizer.cpp
        src/layerPart.cpp
        src/LayerPlan.cpp
        src/LayerPlanBuffer.cpp
        src/mesh.cpp
        src/MeshGroup.cpp
        src/MeshMaterialSplitter.cpp
        src/Mold.cpp
        src/multiVolumes.cpp
        src/path_ordering.cpp
        src/PathAdapter.cpp
        src/PathOrderMonotonic.cpp
        src/Preheat.cpp
        src/PrimeTower/PrimeTower.cpp
        src/PrimeTower/PrimeTowerNormal.cpp
        src/PrimeTower/PrimeTowerInterleaved.cpp
        src/raft.cpp
        src/Scene.cpp
        src/SkeletalTrapezoidation.cpp
        src/SkeletalTrapezoidationGraph.cpp
        src/skin.cpp
        src/SkirtBrim.cpp
        src/SupportInfillPart.cpp
        src/Slice.cpp
        src/sliceDataStorage.cpp
        src/slicer.cpp
        src/support.cpp
        src/timeEstimate.cpp
        src/SlicedUVCoordinates.cpp
        src/TextureDataProvider.cpp
        src/TopSurface.cpp
        src/TreeSupportTipGenerator.cpp
        src/TreeModelVolumes.cpp
        src/TreeSupport.cpp
        src/WallsComputation.cpp
        src/WallToolPaths.cpp

        src/BeadingStrategy/BeadingStrategy.cpp
        src/BeadingStrategy/BeadingStrategyFactory.cpp
        src/BeadingStrategy/DistributedBeadingStrategy.cpp
        src/BeadingStrategy/LimitedBeadingStrategy.cpp
        src/BeadingStrategy/RedistributeBeadingStrategy.cpp
        src/BeadingStrategy/WideningBeadingStrategy.cpp
        src/BeadingStrategy/OuterWallInsetBeadingStrategy.cpp

        src/bridge/bridge.cpp
        src/bridge/ExpansionRange.cpp
        src/bridge/SegmentOverlappingData.cpp
        src/bridge/TransformedSegment.cpp
        src/bridge/TransformedShape.cpp

        src/communication/ArcusCommunication.cpp
        src/communication/ArcusCommunicationPrivate.cpp
        src/communication/CommandLine.cpp
        src/communication/EmscriptenCommunication.cpp
        src/communication/Listener.cpp

        src/infill/ImageBasedDensityProvider.cpp
        src/infill/NoZigZagConnectorProcessor.cpp
        src/infill/ZigzagConnectorProcessor.cpp
        src/infill/LightningDistanceField.cpp
        src/infill/LightningGenerator.cpp
        src/infill/LightningLayer.cpp
        src/infill/LightningTreeNode.cpp
        src/infill/SierpinskiFill.cpp
        src/infill/SierpinskiFillProvider.cpp
        src/infill/SubDivCube.cpp
        src/infill/GyroidInfill.cpp

        src/pathPlanning/Comb.cpp
        src/pathPlanning/GCodePath.cpp
        src/pathPlanning/LinePolygonsCrossings.cpp
        src/pathPlanning/NozzleTempInsert.cpp
        src/pathPlanning/SpeedDerivatives.cpp

        src/plugins/converters.cpp

        src/progress/Progress.cpp
        src/progress/ProgressStageEstimator.cpp

        src/settings/AdaptiveLayerHeights.cpp
        src/settings/FlowTempGraph.cpp
        src/settings/MeshPathConfigs.cpp
        src/settings/PathConfigStorage.cpp
        src/settings/Settings.cpp
        src/settings/ZSeamConfig.cpp

        src/utils/AABB.cpp
        src/utils/AABB3D.cpp
        src/utils/channel.cpp
        src/utils/Date.cpp
        src/utils/ExtrusionJunction.cpp
        src/utils/ExtrusionLine.cpp
        src/utils/ExtrusionSegment.cpp
        src/utils/gettime.cpp
        src/utils/linearAlg2D.cpp
        src/utils/ListPolyIt.cpp
        src/utils/Matrix4x3D.cpp
        src/utils/MeshUtils.cpp
        src/utils/MinimumSpanningTree.cpp
        src/utils/OBJ.cpp
        src/utils/ParameterizedSegment.cpp
        src/utils/PolygonConnector.cpp
        src/utils/PolygonsPointIndex.cpp
        src/utils/PolygonsSegmentIndex.cpp
        src/utils/polygonUtils.cpp
        src/utils/PolylineStitcher.cpp
        src/utils/Simplify.cpp
        src/utils/SVG.cpp
        src/utils/SpatialLookup.cpp
        src/utils/SquareGrid.cpp
        src/utils/ThreadPool.cpp
        src/utils/ToolpathVisualizer.cpp
        src/utils/VoronoiUtils.cpp
        src/utils/VoxelGrid.cpp
        src/utils/VoxelUtils.cpp
        src/utils/MixedPolylineStitcher.cpp

        src/utils/scoring/BestElementFinder.cpp
        src/utils/scoring/CornerScoringCriterion.cpp
        src/utils/scoring/DistanceScoringCriterion.cpp
        src/utils/scoring/ExclusionAreaScoringCriterion.cpp
        src/utils/scoring/PositionBasedScoringCriterion.cpp
        src/utils/scoring/RandomScoringCriterion.cpp
        src/utils/scoring/TextureScoringCriterion.cpp

        src/geometry/Point2LL.cpp
        src/geometry/Point3LL.cpp
        src/geometry/Polygon.cpp
        src/geometry/Shape.cpp
        src/geometry/PointMatrix.cpp
        src/geometry/PointsSet.cpp
        src/geometry/SingleShape.cpp
        src/geometry/PartsView.cpp
        src/geometry/LinesSet.cpp
        src/geometry/Polyline.cpp
        src/geometry/ClosedPolyline.cpp
        src/geometry/MixedLinesSet.cpp

        src/geometry/conversions/Point2D_Point2LL.cpp
)

add_library(_CuraEngine STATIC ${engine_SRCS} ${engine_PB_SRCS})
if (ENABLE_THREADING)
    use_threads(_CuraEngine)
endif ()

target_include_directories(_CuraEngine
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}> # Include Cura.pb.h
)

target_compile_definitions(_CuraEngine
        PUBLIC
        $<$<BOOL:${ENABLE_ARCUS}>:ARCUS>
        $<$<BOOL:${ENABLE_PLUGINS}>:ENABLE_PLUGINS>
        $<$<AND:$<BOOL:${ENABLE_PLUGINS}>,$<BOOL:${ENABLE_REMOTE_PLUGINS}>>:ENABLE_REMOTE_PLUGINS>
        $<$<BOOL:${OLDER_APPLE_CLANG}>:OLDER_APPLE_CLANG>
        CURA_ENGINE_VERSION=\"${CURA_ENGINE_VERSION}\"
        CURA_ENGINE_HASH=\"${CURA_ENGINE_HASH}\"
        $<$<BOOL:${ENABLE_TESTING}>:BUILD_TESTS>
        PRIVATE
        $<$<BOOL:${WIN32}>:NOMINMAX>
        $<$<CONFIG:Debug>:ASSERT_INSANE_OUTPUT>
        $<$<CONFIG:Debug>:USE_CPU_TIME>
        $<$<CONFIG:Debug>:DEBUG>
)

enable_sanitizers(_CuraEngine)

if (${EXTENSIVE_WARNINGS})
    set_project_warnings(_CuraEngine)
endif ()

if (ENABLE_ARCUS)
    target_link_libraries(_CuraEngine PUBLIC arcus::arcus)
endif ()

# Set version variables (normally set by Conan)
if(NOT DEFINED CURA_ENGINE_VERSION)
    set(CURA_ENGINE_VERSION "5.12.0-alpha.0")
endif()
if(NOT DEFINED CURA_ENGINE_HASH)
    find_program(GIT_EXECUTABLE git)
    if(GIT_EXECUTABLE)
        execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            OUTPUT_VARIABLE GIT_HASH
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET)
        if(GIT_HASH)
            set(CURA_ENGINE_HASH ${GIT_HASH})
        else()
            set(CURA_ENGINE_HASH "unknown")
        endif()
    else()
        set(CURA_ENGINE_HASH "unknown")
    endif()
endif()

# mapbox-geometry - vendored in thirdparty/mapbox-geometry
add_library(mapbox_geometry INTERFACE)
target_include_directories(mapbox_geometry INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/mapbox-geometry/include
)
add_library(mapbox::geometry ALIAS mapbox_geometry)
message(STATUS "Using vendored mapbox-geometry from thirdparty/mapbox-geometry")

# mapbox-wagyu - vendored in thirdparty/mapbox-wagyu
add_library(mapbox_wagyu INTERFACE)
target_include_directories(mapbox_wagyu INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/mapbox-wagyu/include
)
target_link_libraries(mapbox_wagyu INTERFACE mapbox::geometry)
add_library(mapbox-wagyu::mapbox-wagyu ALIAS mapbox_wagyu)
message(STATUS "Using vendored mapbox-wagyu from thirdparty/mapbox-wagyu")

# Clipper (polyclipping) - vendored in thirdparty/polyclipping
# Build as static library from vendored sources
add_library(polyclipping STATIC
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/polyclipping/clipper.cpp
)
target_include_directories(polyclipping PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty
)
# Create alias for compatibility with existing code
add_library(clipper::clipper ALIAS polyclipping)
message(STATUS "Building polyclipping from vendored sources")

# RapidJSON - vendored in thirdparty/rapidjson (header-only)
add_library(rapidjson INTERFACE)
target_include_directories(rapidjson INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/rapidjson/include
)
add_library(rapidjson::rapidjson ALIAS rapidjson)
add_library(RapidJSON::RapidJSON ALIAS rapidjson)
message(STATUS "Using vendored rapidjson from thirdparty/rapidjson")

find_package(Boost REQUIRED)
find_package(spdlog REQUIRED)
find_package(fmt REQUIRED)
find_package(range-v3 REQUIRED)

# stb - header-only library
set(stb_INCLUDE_DIRS "/tmp/stb")
if(EXISTS "${stb_INCLUDE_DIRS}/stb_image.h")
    message(STATUS "Found stb at ${stb_INCLUDE_DIRS}")
    add_library(stb::stb INTERFACE IMPORTED)
    set_target_properties(stb::stb PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${stb_INCLUDE_DIRS}"
    )
else()
    message(FATAL_ERROR "stb not found at ${stb_INCLUDE_DIRS}/stb_image.h")
endif()

# scripta - vendored in thirdparty/scripta
# Scripta is header-only, create INTERFACE library
add_library(scripta INTERFACE)
target_include_directories(scripta INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/scripta/include
)
add_library(scripta::scripta ALIAS scripta)
message(STATUS "Using vendored scripta from thirdparty/scripta")
if (ENABLE_ARCUS)
    find_package(PNG REQUIRED)
endif ()
if (NOT EMSCRIPTEN)
    find_package(TBB REQUIRED)
endif ()

if (ENABLE_SENTRY)
    find_package(sentry REQUIRED)
endif ()

if (ENABLE_TESTING)
    find_package(GTest REQUIRED)
endif ()

target_link_libraries(_CuraEngine
        PUBLIC
        spdlog::spdlog
        range-v3::range-v3
        fmt::fmt
        clipper::clipper
        mapbox-wagyu::mapbox-wagyu
        rapidjson
        stb::stb
        Boost::boost
        scripta::scripta
        $<$<BOOL:${ENABLE_ARCUS}>:PNG::PNG>
        $<$<NOT:$<STREQUAL:${CMAKE_SYSTEM_NAME},Emscripten>>:TBB::tbb>
        $<$<TARGET_EXISTS:semver::semver>:semver::semver>
        $<$<TARGET_EXISTS:curaengine_grpc_definitions::curaengine_grpc_definitions>:curaengine_grpc_definitions::curaengine_grpc_definitions>
        $<$<TARGET_EXISTS:asio-grpc::asio-grpc>:asio-grpc::asio-grpc>
        $<$<TARGET_EXISTS:grpc::grpc>:grpc::grpc>
        $<$<TARGET_EXISTS:protobuf::libprotobuf>:protobuf::libprotobuf>
        $<$<TARGET_EXISTS:sentry::sentry>:sentry::sentry>
        $<$<TARGET_EXISTS:GTest::gtest>:GTest::gtest>)

target_compile_definitions(_CuraEngine PRIVATE
        $<$<BOOL:${ENABLE_SENTRY}>:SENTRY_URL=\"${SENTRY_URL}\">
        SENTRY_ENVIRONMENT=\"${SENTRY_ENVIRONMENT}\"
)

if (NOT WIN32)
    add_executable(CuraEngine src/main.cpp) # Then compile main.cpp as separate executable, and link the library to it.
else ()
    message(STATUS "Using windres")
    set(RES_FILES "CuraEngine.rc")
    ENABLE_LANGUAGE(RC)
    if (NOT MSVC)
        SET(CMAKE_RC_COMPILER_INIT windres)
        SET(CMAKE_RC_COMPILE_OBJECT
                "<CMAKE_RC_COMPILER> <FLAGS> -O coff <DEFINES> -i <SOURCE> -o <OBJECT>"
        )
    endif ()
    add_executable(CuraEngine src/main.cpp ${RES_FILES}) # ..., but don't forget the glitter!
    if (ENABLE_SENTRY)
        set_target_properties(CuraEngine PROPERTIES LINK_FLAGS "/DEBUG:FULL")
    endif ()
endif (NOT WIN32)

if (ENABLE_THREADING)
    use_threads(CuraEngine)
endif ()

if (EMSCRIPTEN)
    message(STATUS "Building for Emscripten")
    target_link_options(_CuraEngine
            PUBLIC
            "SHELL:-sINVOKE_RUN=0"
            "SHELL:-sEXPORT_NAME=CuraEngine"
            "SHELL:-sEXPORTED_RUNTIME_METHODS=[callMain,FS]"
            "SHELL:-sFORCE_FILESYSTEM=1"
            "SHELL:-sALLOW_MEMORY_GROWTH=1"
            "SHELL:-sEXPORT_ES6=1"
            "SHELL:-sMODULARIZE=1"
            "SHELL:-sSINGLE_FILE=1"
            "SHELL:-sENVIRONMENT=web"
            "SHELL:-sERROR_ON_UNDEFINED_SYMBOLS=0"
            "SHELL:-sWASM_BIGINT=1"
            "SHELL:-sSTACK_SIZE=196608"
            $<$<CONFIG:Debug>:SHELL:-sASSERTIONS=2>
            $<$<CONFIG:Debug>:SHELL:-sSAFE_HEAP=1>
            $<$<CONFIG:Debug>:SHELL:-sSTACK_OVERFLOW_CHECK=2>
            $<$<CONFIG:Debug>:SHELL:-g3>
            $<$<CONFIG:Debug>:SHELL:-gsource-map>
            "SHELL:-lembind"
            "SHELL:--emit-tsd CuraEngine.d.ts"
    )
endif ()

target_link_libraries(CuraEngine PRIVATE
        _CuraEngine
        $<$<TARGET_EXISTS:sentry::sentry>:sentry::sentry>
)
target_compile_definitions(CuraEngine PRIVATE
        $<$<BOOL:${ENABLE_SENTRY}>:SENTRY_URL=\"${SENTRY_URL}\">
        SENTRY_ENVIRONMENT=\"${SENTRY_ENVIRONMENT}\"
        VERSION=\"${CURA_ENGINE_VERSION}\"
)

# Compiling the test environment.
if (ENABLE_TESTING OR ENABLE_BENCHMARKS)
    set(TESTS_HELPERS_SRC tests/ReadTestPolygons.cpp)

    set(TESTS_SRC_ARCUS)
    if (ENABLE_ARCUS)
        list(APPEND TESTS_SRC_ARCUS
                ArcusCommunicationTest
                ArcusCommunicationPrivateTest)
        list(APPEND TESTS_HELPERS_SRC tests/arcus/MockSocket.cpp)
    endif ()

    add_library(test_helpers ${TESTS_HELPERS_SRC})
    target_compile_definitions(test_helpers PUBLIC $<$<BOOL:${BUILD_TESTING}>:BUILD_TESTS> $<$<BOOL:${ENABLE_ARCUS}>:ARCUS>)
    target_include_directories(test_helpers PUBLIC "include" ${CMAKE_BINARY_DIR}/generated)
    target_link_libraries(test_helpers PRIVATE
            _CuraEngine
            GTest::gtest
            GTest::gmock
            clipper::clipper
            mapbox-wagyu::mapbox-wagyu
            $<$<TARGET_EXISTS:curaengine_grpc_definitions::curaengine_grpc_definitions>:curaengine_grpc_definitions::curaengine_grpc_definitions>
            $<$<TARGET_EXISTS:asio-grpc::asio-grpc>:asio-grpc::asio-grpc>
            $<$<TARGET_EXISTS:grpc::grpc>:grpc::grpc>
            $<$<TARGET_EXISTS:protobuf::libprotobuf>:protobuf::libprotobuf>)
    if (ENABLE_ARCUS)
        target_link_libraries(test_helpers PUBLIC arcus::arcus)
    endif ()
endif ()

if (ENABLE_BENCHMARKS)
    add_subdirectory(benchmark)
    if (NOT WIN32)
        add_subdirectory(stress_benchmark)
    endif ()
endif ()

if (ENABLE_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif ()
