find_package(SteamLanguageParser REQUIRED)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/generated)
execute_process(COMMAND ${SteamLanguageParser_PATH}
    ${SteamKit_DIR}/Resources/SteamLanguage/steammsg.steamd
    -o ${CMAKE_CURRENT_SOURCE_DIR}/generated
    WORKING_DIRECTORY ${SteamKit_DIR}/Resources
    RESULT_VARIABLE result
)

NewTargetSource()
AddSourceFolder(RECURSE INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/generated")

source_group(TREE ${PROJECT_SOURCE_DIR} FILES ${SourceFiles})

set(TARGET_NAME steam_language)
add_library(${TARGET_NAME} INTERFACE ${SourceFiles})
set_target_properties(${TARGET_NAME} PROPERTIES FOLDER "steam")
target_compile_features(${TARGET_NAME} INTERFACE cxx_std_20)
AddTargetInclude(${TARGET_NAME})
AddTargetInstall(${TARGET_NAME} UTILPP)
