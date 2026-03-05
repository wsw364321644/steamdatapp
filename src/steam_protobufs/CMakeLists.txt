

set(ProtobufCPPOutPath ${CMAKE_CURRENT_SOURCE_DIR}/generated)
file(MAKE_DIRECTORY ${ProtobufCPPOutPath})

# if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/protos.csv")
# file(DOWNLOAD
# https://raw.githubusercontent.com/SteamRE/SteamKit/157eb2b01a518bb75fa74c4e2a22dbab181deb11/Resources/ProtobufGen/protos.csv
# ${CMAKE_CURRENT_BINARY_DIR}/protos.csv
# SHOW_PROGRESS
# STATUS DOWNLOAD_STATUS
# )
# list(GET DOWNLOAD_STATUS 0 STATUS_CODE)
# list(GET DOWNLOAD_STATUS 1 ERROR_MESSAGE)

# if(${STATUS_CODE} EQUAL 0)
# message(STATUS "Download completed successfully!")
# else()
# message(FATAL_ERROR "Error occurred during download: ${ERROR_MESSAGE}")
# endif()
# endif()
find_package(Protobuf REQUIRED)
set(Protobuf_INCLUDE ${Protobuf_DIR}/../../../include)
cmake_path(NORMAL_PATH Protobuf_INCLUDE)
set(Protobuf_protoc_PATH ${Protobuf_DIR}/../../..//bin/protoc.exe)
cmake_path(NORMAL_PATH Protobuf_protoc_PATH)

set(OLD_PATH $ENV{PATH})
find_package(absl REQUIRED)
set(absl_PATH ${absl_DIR}/../../../bin)
cmake_path(NORMAL_PATH absl_PATH)
find_package(ZLIB)

if(ZLIB_FOUND)
    set(ZLIB_PATH ${ZLIB_INCLUDE_DIR}/../bin)
    cmake_path(NORMAL_PATH ZLIB_PATH)
    set(ENV{PATH} "${ZLIB_PATH};$ENV{PATH}")
endif()

set(ENV{PATH} "${absl_PATH};$ENV{PATH}")

set(ProtoDirIndex 0)
set(ProtoFileNameIndex 1)
set(ClassFilePathIndex 2)
set(NamespaceIndex 3)

file(READ ${SteamKit_DIR}/Resources/ProtobufGen/protos.csv protos_csv_data)
string(REPLACE "\n" ";" protos_csv_data "${protos_csv_data}")
list(POP_FRONT protos_csv_data)
list(POP_BACK protos_csv_data)

foreach(line IN LISTS protos_csv_data)
    string(REPLACE "," ";" line "${line}")
    list(GET line ${ProtoDirIndex} ProtoDir)
    list(GET line ${ProtoFileNameIndex} ProtoFileName)
    list(APPEND ProtobufFiles ${SteamDBProtobufs_DIR}/${ProtoDir}/${ProtoFileName})
    list(APPEND ProtobufOutFiles ${CMAKE_CURRENT_BINARY_DIR}/${ProtoDir}/${ProtoFileName})
endforeach()

set(INDEX 0)

# fix file content
while(TRUE)
    list(LENGTH ProtobufFiles ProtobufFilesLength)

    if(INDEX EQUAL ProtobufFilesLength OR INDEX GREATER ProtobufFilesLength)
        break()
    endif()

    list(GET ProtobufFiles ${INDEX} FILE)
    list(GET ProtobufOutFiles ${INDEX} OutFile)

    if(NOT EXISTS ${FILE})
        message(WARNING "Protobuf file ${FILE} does not exist, skipping.")
        list(REMOVE_AT ProtobufFiles ${INDEX})
        list(REMOVE_AT ProtobufOutFiles ${INDEX})
        continue()
    endif()

    # ################################  skip google files
    if(${FILE} MATCHES "(.*)google(.*)" AND ${FILE} MATCHES "(.*)protobuf(.*)")
        list(REMOVE_ITEM ProtobufFiles ${FILE})
        continue()
    endif()

    MATH(EXPR INDEX "${INDEX}+1")
    cmake_path(GET FILE PARENT_PATH ProtobufFileParentPath)
    cmake_path(
        RELATIVE_PATH ProtobufFileParentPath
        BASE_DIRECTORY ${SteamDBProtobufs_DIR}
        OUTPUT_VARIABLE ProtobufFileRelativePath
    )

    unset(FILE_CONTENTS_SET)
    file(READ ${FILE} FILE_CONTENTS)

    # ################################ fix import path
    set(regex [=[import \"[^"]+\"\;]=])
    unset(MATCH_OUT)
    string(REGEX MATCHALL ${regex} MATCH_OUT "${FILE_CONTENTS}")

    foreach(MATCH ${MATCH_OUT})
        string(FIND ${MATCH} "\"google/" pos)

        if(NOT pos EQUAL -1)
            continue()
        endif()

        string(REGEX MATCH "\"([^\"]+)\"" MATCH_RESULT ${MATCH})
        string(FIND ${MATCH_RESULT} "${ProtobufFileRelativePath}/" pos)

        if(NOT pos EQUAL -1)
            cmake_path(
                APPEND SteamDBProtobufs_DIR
                ${CMAKE_MATCH_1}
                OUTPUT_VARIABLE IMPORT_FILE_PATH
            )
        else()
            cmake_path(
                APPEND SteamDBProtobufs_DIR
                ${ProtobufFileRelativePath}
                ${CMAKE_MATCH_1}
                OUTPUT_VARIABLE IMPORT_FILE_PATH
            )
            cmake_path(
                APPEND ProtobufFileRelativePath
                ${CMAKE_MATCH_1}
                OUTPUT_VARIABLE IMPORT_PATH
            )
            string(REPLACE ${CMAKE_MATCH_1} ${IMPORT_PATH} FINAL_IMPORT_STR "${MATCH}")
            string(REPLACE ${MATCH} ${FINAL_IMPORT_STR} FILE_CONTENTS "${FILE_CONTENTS}")

            # string(FIND ${MATCH} "\"" pos)
            # math(EXPR pos "${pos}+1" OUTPUT_FORMAT DECIMAL)
            # string(SUBSTRING ${MATCH} 0 ${pos} FIRST_PART)
            # string(SUBSTRING ${MATCH} ${pos} -1 SECOND_PART)
            # string(CONCAT FINAL_IMPORT_STR ${FIRST_PART} "${ProtobufFileRelativePath}/" ${SECOND_PART})
            # string(REPLACE ${MATCH} ${FINAL_IMPORT_STR} FILE_CONTENTS "${FILE_CONTENTS}")
            set(FILE_CONTENTS_SET TRUE)
        endif()

        list(FIND ProtobufFiles ${IMPORT_FILE_PATH} pos)

        if(pos EQUAL -1)
            list(APPEND ProtobufFiles ${IMPORT_FILE_PATH})
            message(STATUS "Adding import file ${IMPORT_FILE_PATH} from ${FILE}.")
            string(REPLACE ${SteamDBProtobufs_DIR} ${CMAKE_CURRENT_BINARY_DIR} IMPORT_FILE_PATH ${IMPORT_FILE_PATH})
            list(APPEND ProtobufOutFiles ${IMPORT_FILE_PATH})
        endif()
    endforeach()

    # ############################## add namespaces
    set(regex [=[^package [^";]+\;]=])
    unset(MATCH_OUT)
    string(REGEX MATCHALL ${regex} MATCH_OUT "${FILE_CONTENTS}")
    list(LENGTH MATCH_OUT list_length)

    if(${list_length} EQUAL 0)
        string(CONCAT FILE_CONTENTS "package utilpp.${ProtobufFileRelativePath};\n" "${FILE_CONTENTS}")
        unset(VAR_LIST)
        set(regex "[ \t\r\n(]\\.[^\";. )]+")
        unset(MATCH_OUT)
        string(REGEX MATCHALL ${regex} MATCH_OUT "${FILE_CONTENTS}")

        foreach(MATCH ${MATCH_OUT})
            string(FIND ${MATCH} ".google" pos)

            if(NOT pos EQUAL -1)
                continue()
            endif()

            string(SUBSTRING ${MATCH} 1 -1 SECOND_PART)
            list(APPEND VAR_LIST ${SECOND_PART})
        endforeach()

        list(REMOVE_DUPLICATES VAR_LIST)

        foreach(VAR ${VAR_LIST})
            string(CONCAT FINAL_VAR ".utilpp.${ProtobufFileRelativePath}" ${VAR})
            string(CONCAT VAR_TEMP ${VAR} " ")
            string(CONCAT FINAL_VAR_TEMP ${FINAL_VAR} " ")
            string(REPLACE ${VAR_TEMP} ${FINAL_VAR_TEMP} FILE_CONTENTS "${FILE_CONTENTS}")
            string(CONCAT VAR_TEMP ${VAR} ".")
            string(CONCAT FINAL_VAR_TEMP ${FINAL_VAR} ".")
            string(REPLACE ${VAR_TEMP} ${FINAL_VAR_TEMP} FILE_CONTENTS "${FILE_CONTENTS}")
            string(CONCAT VAR_TEMP ${VAR} ")")
            string(CONCAT FINAL_VAR_TEMP ${FINAL_VAR} ")")
            string(REPLACE ${VAR_TEMP} ${FINAL_VAR_TEMP} FILE_CONTENTS "${FILE_CONTENTS}")
        endforeach()

        set(FILE_CONTENTS_SET TRUE)
    endif()

    # ############################## fix proto3
    string(FIND "${FILE_CONTENTS}" [=[syntax = "proto3";]=] pos)

    if(NOT pos EQUAL -1)
        set(regex [=[\[default = [^]]+\]]=])
        string(REGEX REPLACE ${regex} "" FILE_CONTENTS "${FILE_CONTENTS}")
        set(FILE_CONTENTS_SET TRUE)
    endif()

    file(WRITE ${OutFile} "${FILE_CONTENTS}")
endwhile()

foreach(FILE ${ProtobufOutFiles})
    message(STATUS "Generating C++ files from ${FILE}")

    execute_process(
        COMMAND ${Protobuf_protoc_PATH}
        --proto_path=${Protobuf_INCLUDE}
        --proto_path=${CMAKE_CURRENT_BINARY_DIR}

        # --proto_path=${ProtobufFileParentPath}
        --cpp_out=${ProtobufCPPOutPath}
        ${FILE}

        # WORKING_DIRECTORY ${Protobuf_PROTOC_PATH}
        OUTPUT_VARIABLE output
        RESULT_VARIABLE result
    )

    if(NOT ${result} EQUAL 0)
        message(FATAL_ERROR "Failed to generate C++ files from ${FILE} COMMAND ${Protobuf_protoc_PATH} --proto_path=${Protobuf_INCLUDE}  --proto_path=${CMAKE_CURRENT_BINARY_DIR} --cpp_out=${ProtobufCPPOutPath}  output: ${output}")
    endif()
endforeach()

set(ENV{PATH} "${OLD_PATH}")

NewTargetSource()
AddSourceFolder(RECURSE PUBLIC "${ProtobufCPPOutPath}")
source_group(TREE ${PROJECT_SOURCE_DIR} FILES ${SourceFiles})

function(add_lib TARGET_NAME)
    set_target_properties(${TARGET_NAME} PROPERTIES FOLDER "steam")

    target_compile_options(${TARGET_NAME} PRIVATE
        $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
        -Wall -Wextra -Wconversion -pedantic-errors -Wsign-conversion>
        $<$<CXX_COMPILER_ID:MSVC>:
        /bigobj /wd4146>
    )

    target_link_libraries(${TARGET_NAME} PUBLIC protobuf::libprotobuf)

    if(NOT DISABLE_INSTALL)
        AddTargetInclude(${TARGET_NAME})
        AddTargetInstall(${TARGET_NAME} ${PROJECT_NAME})
    endif()
endfunction()

set(TARGET_NAME steam_protobufs_a)
add_library(${TARGET_NAME} STATIC ${SourceFiles})
add_lib(${TARGET_NAME})

set(TARGET_NAME steam_protobufs)
add_library(${TARGET_NAME} SHARED ${SourceFiles})
add_lib(${TARGET_NAME})