cmake_minimum_required (VERSION 3.12.4)
project (pe-sieve)

option(PESIEVE_AS_DLL "Build PE-sieve as a DLL" OFF)
option(PESIEVE_AS_STATIC_LIB "Build PE-Sieve as a static library" OFF)

include_directories (
	include
)
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")

	option(LINK_STATICALLY "Link PE-Sieve with static versions of linked libraries" ON)

	add_compile_options(
		-fpermissive
	)

	add_compile_definitions(
		_WIN32_WINNT=1536 # 0x600 aka Windows Vista required
	)

	if(LINK_STATICALLY)
		add_link_options(
			-static
			-static-libgcc
			-static-libstdc++
		)
	endif()

endif()

set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")

# modules:
set ( M_PARSER "libpeconv/libpeconv" )

# modules paths:
set (PECONV_DIR "${PROJECT_SOURCE_DIR}/${M_PARSER}" CACHE PATH "PEConv main path")
include_directories ( ${PECONV_DIR}/include )

set (scanners_srcs
	scanners/mapping_scanner.cpp
	scanners/headers_scanner.cpp
	scanners/code_scanner.cpp
	scanners/workingset_scanner.cpp
	scanners/artefact_scanner.cpp
	scanners/iat_scanner.cpp
	scanners/scanner.cpp
	scanners/module_data.cpp
	scanners/mempage_data.cpp
	scanners/patch_list.cpp
	scanners/patch_analyzer.cpp
	scanners/scan_report.cpp
	scanners/scanned_modules.cpp
	scanners/hook_targets_resolver.cpp
)

set (scanners_hdrs
	scanners/mapping_scanner.h
	scanners/headers_scanner.h
	scanners/code_scanner.h
	scanners/module_scanner.h
	scanners/workingset_scanner.h
	scanners/artefact_scanner.h
	scanners/iat_scanner.h
	scanners/scanner.h
	scanners/module_scan_report.h
	scanners/scan_report.h
	scanners/module_data.h
	scanners/mempage_data.h
	scanners/pe_section.h
	scanners/patch_list.h
	scanners/patch_analyzer.h
	scanners/scanned_modules.h
	scanners/hook_targets_resolver.h
)

set (imprec_hdrs
	postprocessors/imp_rec/imp_reconstructor.h
	postprocessors/imp_rec/iat_block.h
	postprocessors/imp_rec/iat_finder.h
	postprocessors/imp_rec/import_table_finder.h
)

set (postprocessors_hdrs
	postprocessors/report_formatter.h
	postprocessors/results_dumper.h
	postprocessors/dump_report.h
	postprocessors/pe_buffer.h
	postprocessors/pe_reconstructor.h
)

set (imprec_srcs
	postprocessors/imp_rec/imp_reconstructor.cpp
	postprocessors/imp_rec/iat_block.cpp
	postprocessors/imp_rec/iat_finder.cpp
	postprocessors/imp_rec/import_table_finder.cpp
)

set (postprocessors_srcs
	postprocessors/report_formatter.cpp
	postprocessors/results_dumper.cpp
	postprocessors/dump_report.cpp
	postprocessors/pe_buffer.cpp
	postprocessors/pe_reconstructor.cpp
)

set (utils_srcs
	utils/format_util.cpp
	utils/process_util.cpp
	utils/process_privilege.cpp
	utils/process_minidump.cpp
	utils/path_util.cpp
	utils/path_converter.cpp
	utils/workingset_enum.cpp
	utils/modules_enum.cpp
	utils/artefacts_util.cpp
	utils/process_reflection.cpp
)

set (utils_hdrs
	utils/format_util.h
	utils/path_util.h
	utils/path_converter.h
	utils/process_util.h
	utils/process_privilege.h
	utils/process_minidump.h
	utils/workingset_enum.h
	utils/modules_enum.h
	utils/artefacts_util.h
	utils/process_reflection.h
)

set (params_info_hdrs
	params_info/pe_sieve_params_info.h
)

set (params_info_srcs
	params_info/pe_sieve_params_info.cpp
)

set (srcs
	pe_sieve.cpp
	color_scheme.h
	${params_info_srcs}
	${scanners_srcs}
	${postprocessors_srcs}
	${imprec_srcs}
	${utils_srcs}
)

set (hdrs
	pe_sieve.h
	include/pe_sieve_types.h
	${params_info_hdrs}
	${scanners_hdrs}
	${postprocessors_hdrs}
	${imprec_hdrs}
	${utils_hdrs}
)

set (rsrc
	resources.h
	resources.rc
)

# set filters in Visual Studio project:

SOURCE_GROUP("Source Files\\scanners" FILES ${scanners_srcs} )
SOURCE_GROUP("Header Files\\scanners" FILES ${scanners_hdrs} )

SOURCE_GROUP("Source Files\\utils" FILES ${utils_srcs} )
SOURCE_GROUP("Header Files\\utils" FILES ${utils_hdrs} )

SOURCE_GROUP("Source Files\\params_info" FILES ${params_info_srcs} )
SOURCE_GROUP("Header Files\\params_info" FILES ${params_info_hdrs} )

SOURCE_GROUP("Source Files\\postprocessors" FILES ${postprocessors_srcs} )
SOURCE_GROUP("Header Files\\postprocessors" FILES ${postprocessors_hdrs} )

SOURCE_GROUP("Source Files\\postprocessors\\imp_rec" FILES ${imprec_srcs} )
SOURCE_GROUP("Header Files\\postprocessors\\imp_rec" FILES ${imprec_hdrs} )

include(GNUInstallDirs)
set (library_export_hdrs
	include/pe_sieve_types.h
	include/pe_sieve_version.h
	include/pe_sieve_api.h
)
if(PESIEVE_AS_STATIC_LIB)
	set(LIBRARY_TYPE STATIC)
	# Install libpeconv as well since with static libraries a user has to link against both
	set(PECONV_LIB_INSTALL ON CACHE INTERNAL "")
elseif(PESIEVE_AS_DLL)
	set(LIBRARY_TYPE SHARED)
endif()

if (PESIEVE_AS_DLL AND PESIEVE_AS_STATIC_LIB)
	message(FATAL_ERROR "Choose either PESIEVE_AS_STATIC_LIB or PESIEVE_AS_DLL (or none of them, to build as an EXE)")
endif()

# Choose to build the library or EXE
if(PESIEVE_AS_STATIC_LIB OR PESIEVE_AS_DLL)
	set (lib_srcs
		pe_sieve_api.cpp
		${srcs}
	)
	if (PESIEVE_AS_DLL)
		set (lib_srcs
			dll_main.cpp
			main.def
			${lib_srcs}
		)
	endif()

	add_library ( ${PROJECT_NAME} ${LIBRARY_TYPE} ${library_export_hdrs} ${hdrs} ${lib_srcs} )
	set_source_files_properties(main.def PROPERTIES HEADER_FILE_ONLY TRUE)
	target_compile_definitions(${PROJECT_NAME} PUBLIC PESIEVE_${LIBRARY_TYPE}_LIB)
	INSTALL(FILES ${library_export_hdrs}
		DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
	)
	set_source_files_properties(main.def PROPERTIES HEADER_FILE_ONLY TRUE)
else()
	add_executable ( ${PROJECT_NAME} ${hdrs} ${srcs} ${rsrc} main.cpp )
endif()

if(PESIEVE_AS_STATIC_LIB OR PESIEVE_AS_DLL)
	include(GNUInstallDirs)
	install(TARGETS ${PROJECT_NAME}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	)
endif()

install(TARGETS ${PROJECT_NAME}
	RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX} COMPONENT ${PROJECT_NAME}
)


# libs
add_subdirectory (libpeconv/libpeconv)
set ( PECONV_LIB $<TARGET_FILE:libpeconv> CACHE FILE "PEConvLib library path" )

target_link_libraries ( ${PROJECT_NAME} ${PECONV_LIB} "psapi.lib" "ntdll.lib" "shlwapi")

# dependencies
add_dependencies(${PROJECT_NAME} libpeconv)
