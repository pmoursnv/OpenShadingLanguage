# Copyright Contributors to the Open Shading Language project.
# SPDX-License-Identifier: BSD-3-Clause
# https://github.com/AcademySoftwareFoundation/OpenShadingLanguage

# The 'testshade' executable
set ( testshade_srcs
      testshade.cpp
      simplerend.cpp )

if (BUILD_BATCHED)
    list(APPEND testshade_srcs 
        batched_simplerend.cpp)
endif()    

if (USE_OPTIX)
    list (APPEND testshade_srcs optixgridrender.cpp)
    set ( testshade_cuda_srcs
        cuda/optix_grid_renderer.cu
        ../testrender/cuda/wrapper.cu
        ../testrender/cuda/rend_lib.cu )

    set ( testshade_cuda_headers
        ../testrender/cuda/rend_lib.h )

    # We need to make sure that the PTX files are regenerated whenever these
    # headers change.
    set ( extra_cuda_headers
        render_params.h )

    LLVM_COMPILE_CUDA (
        ${CMAKE_CURRENT_SOURCE_DIR}/../testrender/cuda/rend_lib.cu
        ${testshade_cuda_headers}
        "rend_llvm_compiled_ops"
        rend_lib_bc_cpp
        "-I../testrender/cuda" )

    list (APPEND testshade_srcs ${rend_lib_bc_cpp})

    # Generate PTX for all of the CUDA files
    foreach (cudasrc ${testshade_cuda_srcs})
        NVCC_COMPILE ( ${cudasrc} ${extra_cuda_headers} ptx_generated "-I../testrender/cuda" )
        list (APPEND ptx_list ${ptx_generated})
    endforeach ()

    add_custom_target (testshade_ptx ALL
        DEPENDS ${ptx_list}
        SOURCES ${testshade_cuda_srcs} )

    # Install the PTX files in a fixed location so that they can be
    # loaded at run time
    install (FILES ${ptx_list}
             DESTINATION ${OSL_PTX_INSTALL_DIR})
endif()

set ( rs_srcs 
    rs_simplerend.cpp )
EMBED_LLVM_BITCODE_IN_CPP ( "${rs_srcs}" "_host" "testshade_llvm_compiled_rs" testshade_srcs "-DOSL_HOST_RS_BITCODE=1")

add_executable ( testshade ${testshade_srcs} testshademain.cpp )

target_link_libraries (testshade
                       PRIVATE
                           oslexec oslquery)

install (TARGETS testshade RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} )

osl_optix_target(testshade)


# As explained in PR #39, problems were reported with calling OSL from a
# Houdini plugin.  So this section sets up a version of testshade as a DSO, to
# be sure we can link it into an executable properly.
#
# But the fact that we use testshade.cpp twice -- once for regular testshade,
# once for the DSO version -- is throwing off the code coverage test, so at
# least for now, disable the dso version entirely when CODECOV is on.
if (NOT CODECOV)
    # The 'libtestshade' library
    add_library ( "libtestshade" ${testshade_srcs} )

    set_target_properties (libtestshade
                           PROPERTIES
                           VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}
                           SOVERSION ${SOVERSION}
                           OUTPUT_NAME libtestshade${OSL_LIBNAME_SUFFIX}
                           )

    target_link_libraries (libtestshade
                           PRIVATE
                               oslexec oslquery)
    set_target_properties (libtestshade PROPERTIES PREFIX "")

    install_targets ( libtestshade )

    # The 'testshade_dso' executable
    add_executable ( testshade_dso testshade_dso.cpp )
    target_link_libraries (testshade_dso
                           PRIVATE
                               OpenImageIO::OpenImageIO
                               ${Boost_LIBRARIES} ${CMAKE_DL_LIBS} )
    install (TARGETS testshade_dso RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} )
    osl_optix_target(libtestshade)
endif ()
