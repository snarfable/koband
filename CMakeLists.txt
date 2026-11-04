cmake_minimum_required(VERSION 3.16)
project(koband NONE)

# =============================================================================
# Configuration & Discovery
# =============================================================================

# Options and settings
set(PKM_INPFILE "packmol-pos.inp" CACHE STRING "Packmol input template name")
set(SIM_PREFIX "" CACHE STRING "Simulation prefix index (optional)")
set(SIM_SUFFIX "" CACHE STRING "Simulation suffix jobid (optional)")
option(BUILD_SIMULATION_WITH_ALL "Run simulation end targets as part of 'all'" OFF)

# Find required tools
find_package(Perl REQUIRED)
find_program(PACKMOL_EXECUTABLE NAMES packmol REQUIRED)
find_program(OPENBABEL_EXECUTABLE NAMES obabel babel REQUIRED)
find_program(GROMACS_EXECUTABLE NAMES gmx gmx_d gmx_mpi)
find_program(BASH_EXECUTABLE NAMES bash REQUIRED)
find_program(CPP_EXECUTABLE NAMES cpp REQUIRED)
find_program(AWK_EXECUTABLE NAMES gawk awk REQUIRED)

# Find optional Tinker tools
find_program(TNK_DYNAMIC NAMES dynamic)
find_program(TNK_MINIMIZE NAMES minimize)

# =============================================================================
# Build Environment Staging
# =============================================================================

# Stage required scripts and static input files into binary directory for out-of-source builds
set(STAGED_FILES
    kaA.pdb
    kaB.pdb
    configure.pl
    topsect
    runtopsect
    gmx-mdrun
)

if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    foreach(FILE_NAME ${STAGED_FILES})
        configure_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/${FILE_NAME}"
            "${CMAKE_CURRENT_BINARY_DIR}/${FILE_NAME}"
            COPYONLY
        )
    endforeach()
endif()

# Ensure executable scripts have execution permissions
file(CHMOD "${CMAKE_CURRENT_BINARY_DIR}/configure.pl"
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
file(CHMOD "${CMAKE_CURRENT_BINARY_DIR}/topsect"
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
file(CHMOD "${CMAKE_CURRENT_BINARY_DIR}/runtopsect"
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
file(CHMOD "${CMAKE_CURRENT_BINARY_DIR}/gmx-mdrun"
    FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

# =============================================================================
# Template Configuration (.tt -> target files)
# =============================================================================

set(TEMPLATE_FILES
    packmol-pos.inp.tt
    packmol.inp.tt
    config.txt.tt
    fixbox.tt
    keyfile.key.tt
    prmfile.prm.tt
    step-min.mdp.tt
    step-pre.mdp.tt
    step-run.mdp.tt
    topol.top.tt
)

foreach(TT_FILE ${TEMPLATE_FILES})
    string(REGEX REPLACE "\\.tt$" "" OUT_FILE "${TT_FILE}")
    set(SRC_TT "${CMAKE_CURRENT_SOURCE_DIR}/${TT_FILE}")
    set(BIN_TT "${CMAKE_CURRENT_BINARY_DIR}/${TT_FILE}")
    set(BIN_OUT "${CMAKE_CURRENT_BINARY_DIR}/${OUT_FILE}")

    if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
        add_custom_command(
            OUTPUT "${BIN_OUT}"
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "${SRC_TT}" "${BIN_TT}"
            COMMAND ${PERL_EXECUTABLE} "${CMAKE_CURRENT_BINARY_DIR}/configure.pl" "${TT_FILE}"
            DEPENDS "${SRC_TT}" "${CMAKE_CURRENT_BINARY_DIR}/configure.pl"
            WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
            COMMENT "Configuring ${OUT_FILE} from ${TT_FILE}"
            VERBATIM
        )
    else()
        add_custom_command(
            OUTPUT "${BIN_OUT}"
            COMMAND ${PERL_EXECUTABLE} "${CMAKE_CURRENT_BINARY_DIR}/configure.pl" "${TT_FILE}"
            DEPENDS "${SRC_TT}" "${CMAKE_CURRENT_BINARY_DIR}/configure.pl"
            WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
            COMMENT "Configuring ${OUT_FILE} from ${TT_FILE}"
            VERBATIM
        )
    endif()
endforeach()

# =============================================================================
# Initial Structure & Simulation Preparation
# =============================================================================

# 1. Packmol initial coordinate generation
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb"
    COMMAND ${BASH_EXECUTABLE} -c "${PACKMOL_EXECUTABLE} < ${PKM_INPFILE}"
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/${PKM_INPFILE}"
            "${CMAKE_CURRENT_BINARY_DIR}/kaA.pdb"
            "${CMAKE_CURRENT_BINARY_DIR}/kaB.pdb"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Generating initial configuration confin.pdb via Packmol"
    VERBATIM
)

add_custom_target(packmol
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb"
    COMMENT "Packmol initial snapshot complete"
)

# 2. GROMACS preparation files
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/confin.gro"
    COMMAND ${OPENBABEL_EXECUTABLE} -ipdb "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb" -ogro -O "${CMAKE_CURRENT_BINARY_DIR}/confin.gro"
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Converting confin.pdb to confin.gro"
    VERBATIM
)

add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/confout-box.gro"
    COMMAND ${BASH_EXECUTABLE} fixbox
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/confin.gro"
            "${CMAKE_CURRENT_BINARY_DIR}/fixbox"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Adjusting box dimensions via fixbox -> confout-box.gro"
    VERBATIM
)

set(GMX_TARGETS
    "${CMAKE_CURRENT_BINARY_DIR}/confin.gro"
    "${CMAKE_CURRENT_BINARY_DIR}/confout-box.gro"
    "${CMAKE_CURRENT_BINARY_DIR}/topol.top"
    "${CMAKE_CURRENT_BINARY_DIR}/step-min.mdp"
    "${CMAKE_CURRENT_BINARY_DIR}/step-pre.mdp"
    "${CMAKE_CURRENT_BINARY_DIR}/step-run.mdp"
)

add_custom_target(gmx-prep
    DEPENDS ${GMX_TARGETS}
    COMMENT "GROMACS preparation files successfully generated"
)

# 3. Tinker preparation files
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/confin.xyz"
    COMMAND ${BASH_EXECUTABLE} -c "${OPENBABEL_EXECUTABLE} -ipdb confin.pdb -otxyz -O confin.xyz.temp && sed -r -e '2,6401s/0\$/1/g' -e '6402,8001s/0\$/2/g' confin.xyz.temp > confin.xyz && rm -f confin.xyz.temp"
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Converting confin.pdb to confin.xyz for Tinker"
    VERBATIM
)

add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/header.tmp"
    COMMAND ${BASH_EXECUTABLE} -c "printf '\\n##############################\\n# This Tinker parameter file #\\n# was generated by topsect   #\\n##############################\\n\\nforcefield       OPLS-AA/L\\n\\nvdwtype          LENNARD-JONES\\nradiusrule       GEOMETRIC\\nradiussize       DIAMETER\\nradiustype       SIGMA\\nepsilonrule      GEOMETRIC\\ntorsionunit      0.5\\nimptorunit       0.5\\nvdw-14-scale     2.0\\nchg-14-scale     2.0\\n\\n' > header.tmp"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Generating Tinker parameter header.tmp"
    VERBATIM
)

add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/output.prm"
    COMMAND ${BASH_EXECUTABLE} -c "${CPP_EXECUTABLE} -I. ./topol.top | ./topsect -"
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/topol.top"
            "${CMAKE_CURRENT_BINARY_DIR}/header.tmp"
            "${CMAKE_CURRENT_BINARY_DIR}/topsect"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    COMMENT "Generating Tinker parameter file output.prm via topsect"
    VERBATIM
)

set(TNK_TARGETS
    "${CMAKE_CURRENT_BINARY_DIR}/confin.xyz"
    "${CMAKE_CURRENT_BINARY_DIR}/keyfile.key"
    "${CMAKE_CURRENT_BINARY_DIR}/output.prm"
)

add_custom_target(tnk-prep
    DEPENDS ${TNK_TARGETS}
    COMMENT "Tinker preparation files successfully generated"
)

# Default all target builds prep targets
add_custom_target(all-prep ALL
    DEPENDS gmx-prep tnk-prep
    COMMENT "All GROMACS and Tinker preparation targets ready"
)

# =============================================================================
# Generalized Simulation Stages (Custom Targets & Commands)
# =============================================================================

set(SIMULATION_STAGES "")

# Function to register and define a simulation stage
function(add_simulation_stage)
    cmake_parse_arguments(ARG "" "STAGE;PREV_STAGE;PREV_DIR" "DEPENDS" ${ARGN})

    set(STAGE_NAME "${ARG_STAGE}")
    set(STAGE_DIR "run-${STAGE_NAME}")
    set(STAGE_TARGET "run-${STAGE_NAME}")
    set(STAGE_OUT "${CMAKE_CURRENT_BINARY_DIR}/${STAGE_DIR}/confout-box-${STAGE_NAME}.gro")

    set(CMD_ARGS "-s ${STAGE_NAME}")
    if(ARG_PREV_STAGE)
        string(APPEND CMD_ARGS " -S ${ARG_PREV_STAGE}")
    endif()
    if(ARG_PREV_DIR)
        string(APPEND CMD_ARGS " -p ${ARG_PREV_DIR}")
    endif()
    if(SIM_PREFIX)
        string(APPEND CMD_ARGS " -i ${SIM_PREFIX}")
    endif()
    if(SIM_SUFFIX)
        string(APPEND CMD_ARGS " -j ${SIM_SUFFIX}")
    endif()

    add_custom_command(
        OUTPUT "${STAGE_OUT}"
        COMMAND ${CMAKE_COMMAND} -E make_directory "${STAGE_DIR}"
        COMMAND ${BASH_EXECUTABLE} -c "cd ${STAGE_DIR} && RESTARTED=0 ../gmx-mdrun ${CMD_ARGS}"
        DEPENDS ${GMX_TARGETS}
                "${CMAKE_CURRENT_BINARY_DIR}/gmx-mdrun"
                ${ARG_DEPENDS}
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
        COMMENT "Executing GROMACS simulation stage '${STAGE_NAME}' -> ${STAGE_OUT}"
        VERBATIM
    )

    add_custom_target(${STAGE_TARGET}
        DEPENDS "${STAGE_OUT}"
        COMMENT "Simulation stage '${STAGE_NAME}' target complete: ${STAGE_OUT}"
    )

    # Shorthand alias target without 'run-' prefix
    if(NOT TARGET "${STAGE_NAME}")
        add_custom_target("${STAGE_NAME}" DEPENDS ${STAGE_TARGET})
    endif()

    set(SIMULATION_STAGES "${SIMULATION_STAGES};${STAGE_TARGET}" PARENT_SCOPE)
    set(${STAGE_TARGET}_OUTPUT "${STAGE_OUT}" PARENT_SCOPE)
endfunction()

# Define simulation pipeline stages
# 1. Minimization
add_simulation_stage(
    STAGE min
)

# 2. Pre-equilibration
add_simulation_stage(
    STAGE pre
    PREV_STAGE min
    PREV_DIR ../run-min
    DEPENDS ${run-min_OUTPUT}
)

# 3. Production Run
add_simulation_stage(
    STAGE run
    PREV_STAGE pre
    PREV_DIR ../run-pre
    DEPENDS ${run-pre_OUTPUT}
)

# Umbrella custom target to execute entire simulation workflow
add_custom_target(simulation
    DEPENDS run-run
    COMMENT "All simulation end targets (run-min -> run-pre -> run-run) completed!"
)

if(BUILD_SIMULATION_WITH_ALL)
    add_custom_target(all-simulation ALL
        DEPENDS simulation
    )
endif()

# =============================================================================
# Utility Targets: Check and Clean
# =============================================================================

add_custom_target(check
    COMMAND ${CMAKE_COMMAND} -E echo "Kob-Andersen Simulation System Configuration:"
    COMMAND ${CMAKE_COMMAND} -E echo "  Packmol Input : ${PKM_INPFILE}"
    COMMAND ${CMAKE_COMMAND} -E echo "  Sim Prefix    : ${SIM_PREFIX}"
    COMMAND ${CMAKE_COMMAND} -E echo "  Sim Suffix    : ${SIM_SUFFIX}"
    COMMAND ${CMAKE_COMMAND} -E echo "GROMACS Preparation Targets:"
    COMMAND ${CMAKE_COMMAND} -E echo "  ${GMX_TARGETS}"
    COMMAND ${CMAKE_COMMAND} -E echo "Tinker Preparation Targets:"
    COMMAND ${CMAKE_COMMAND} -E echo "  ${TNK_TARGETS}"
    COMMAND ${CMAKE_COMMAND} -E echo "Simulation End Targets:"
    COMMAND ${CMAKE_COMMAND} -E echo "  run-min, run-pre, run-run (or 'simulation')"
    VERBATIM
)

set_property(DIRECTORY PROPERTY ADDITIONAL_CLEAN_FILES
    "${CMAKE_CURRENT_BINARY_DIR}/output.log"
    "${CMAKE_CURRENT_BINARY_DIR}/confin.pdb"
    "${CMAKE_CURRENT_BINARY_DIR}/confin.gro"
    "${CMAKE_CURRENT_BINARY_DIR}/confout-box.gro"
    "${CMAKE_CURRENT_BINARY_DIR}/confin.xyz"
    "${CMAKE_CURRENT_BINARY_DIR}/confin.xyz.temp"
    "${CMAKE_CURRENT_BINARY_DIR}/header.tmp"
    "${CMAKE_CURRENT_BINARY_DIR}/output.prm"
    "${CMAKE_CURRENT_BINARY_DIR}/fixbox"
    "${CMAKE_CURRENT_BINARY_DIR}/run-min"
    "${CMAKE_CURRENT_BINARY_DIR}/run-pre"
    "${CMAKE_CURRENT_BINARY_DIR}/run-run"
)

add_custom_target(clean-simulation
    COMMAND ${CMAKE_COMMAND} -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/run-min"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/run-pre"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/run-run"
    COMMENT "Removed simulation run directories"
)
