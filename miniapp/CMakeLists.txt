################
#  Build test  #
################
set(executables "cosma_miniapp" "cosma_statistics" "scalars_miniapp")

foreach(exec ${executables})
    add_executable(${exec} "${exec}.cpp")
    target_link_libraries(${exec} PRIVATE cosma options)
endforeach()

if(COSMA_SCALAPACK)
    set(scalapack_executables "pdgemm_miniapp")
    foreach(exec ${scalapack_executables})
        add_executable(${exec} "${exec}.cpp")
        target_link_libraries(${exec} PRIVATE cosma_pxgemm_cpp options)
        if (COSMA_WITH_PROFILING)
            target_link_libraries(${exec} PRIVATE semiprof)
            target_compile_definitions(${exec} PRIVATE COSMA_WITH_PROFILING)
        endif()
    endforeach()
endif()
