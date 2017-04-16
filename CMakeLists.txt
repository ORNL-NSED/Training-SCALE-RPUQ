CMAKE_MINIMUM_REQUIRED( VERSION 2.8 )

PROJECT( "Training-SCALE-RPUQ"
    LANGUAGES NONE
)

IF( "${SCALERTE}" STREQUAL "" )
    MESSAGE( FATAL_ERROR "variable SCALERTE must be set!!!" )
ENDIF()
IF( "${N_SAMPLES}" STREQUAL "" )
    SET(N_SAMPLES 100)
ENDIF()
IF( "${N_THREADS}" STREQUAL "" )
    INCLUDE( ProcessorCount )
    ProcessorCount( N_THREADS )
ENDIF()
IF( "${DIR_MATCH}" STREQUAL "" )
    SET(DIR_MATCH "/solutions")
ENDIF()
FILE(GLOB_RECURSE inputs RELATIVE "${CMAKE_SOURCE_DIR}" src/*.inp)
FOREACH( input ${inputs} )
    GET_FILENAME_COMPONENT( dir "${input}" DIRECTORY )
    IF( "${dir}" MATCHES "${DIR_MATCH}" )

        #copy over input file
        GET_FILENAME_COMPONENT( file "${input}" NAME )
        GET_FILENAME_COMPONENT( basename "${file}" NAME_WE )
        SET( inpdir "${CMAKE_BINARY_DIR}/${dir}")

        FILE(READ "${input}" contents )
        #replace N_SAMPLES
        STRING(REGEX REPLACE "n_samples *= *[0-9]+" "n_samples = ${N_SAMPLES}" contents ${contents})

        SET(basename_list ${basename})

        #directive to create two input files that have perturb_xs=yes/no
        #directive to create two input files that have fp=yes/no
        FOREACH( mod "xs" "fp" )
            SET(new_list)
            FOREACH( b ${basename_list} )
                IF( "${b}" MATCHES "~${mod}" )
                    STRING(REGEX REPLACE "~${mod}" "-${mod}" b1 ${b} )
                    STRING(REGEX REPLACE "~${mod}" "" b2 ${b} )
                    LIST(APPEND new_list ${b1} ${b2})
                ELSE()
                    LIST(APPEND new_list ${b})
                ENDIF()
            ENDFOREACH()
            SET(basename_list ${new_list})
        ENDFOREACH()
        SET(new_list)
        FOREACH( b ${basename_list} )
            IF( NOT "${b}" MATCHES "~" )
                LIST(APPEND new_list ${b})
            ENDIF()
        ENDFOREACH()
        SET(basename_list ${new_list})
        #come out with a new list

        FOREACH( b ${basename_list} )

            #change perturb_xs if original basename has ~xs
            IF( ${b} MATCHES "-xs" )
                STRING(REGEX REPLACE "perturb_xs *= *(yes|no)" "perturb_xs = yes" contents ${contents})
            ELSE()
                STRING(REGEX REPLACE "perturb_xs *= *(yes|no)" "perturb_xs = no" contents ${contents})
            ENDIF()

            #change perturb_yields if original basename has ~fp
            IF( ${b} MATCHES "-fp" )
                STRING(REGEX REPLACE "perturb_yields *= *(yes|no)" "perturb_yields = yes" contents ${contents})
            ELSE()
                STRING(REGEX REPLACE "perturb_yields *= *(yes|no)" "perturb_yields = no" contents ${contents})
            ENDIF()


            #write file
            FILE(WRITE "${inpdir}/${b}.inp" ${contents} )

            #run scale
            ADD_CUSTOM_COMMAND(
                OUTPUT "${inpdir}/${b}.out"
                DEPENDS "${inpdir}/${b}.inp"
                COMMAND "${SCALERTE}" ARGS -m -I ${N_THREADS} "${b}.inp"
                WORKING_DIRECTORY "${inpdir}"
            )
            ADD_CUSTOM_TARGET( "${b}" ALL DEPENDS "${inpdir}/${b}.out" )

        ENDFOREACH()
    ENDIF()
ENDFOREACH()
