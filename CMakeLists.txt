cmake_minimum_required(VERSION 3.21)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
project(embed LANGUAGES CXX)

# Options
option(B_EMBED_BUILD_EXAMPLES "Build battery::embed examples" ${PROJECT_IS_TOP_LEVEL})
option(B_PRODUCTION_MODE "Enable production mode and disable development tools such as hotreload" OFF)
option(B_EMBED_SILENCE_DEVMODE_WARNING "Disable the battery::embed devmode warning" OFF)

# Remember the binary dir for later
set(EMBED_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/embed CACHE INTERNAL "binary directory of the battery::embed library" FORCE)

# The template for how to generate the .cpp file
set(EMBED_SOURCE_FILE_TEMPLATE [==[
// File generated using battery::embed (https://github.com/batterycenter/embed)
// Embedded file: '${FILENAME}' as '${IDENTIFIER}'
// Filesize: ${FILESIZE} bytes
// DO NOT EDIT THIS FILE!!!

#include "battery/embed.hpp"

namespace b {
    const unsigned char ${IDENTIFIER}_data[] = {
        ${GENERATED_BYTE_ARRAY}
    };
    EmbedInternal::EmbeddedFile EmbedInternal::${IDENTIFIER} = {
            ${IDENTIFIER}_data,
            ${FILESIZE}
        , "${FILENAME}"
#ifndef B_PRODUCTION_MODE
        , "${FULL_PATH}"
#endif // !B_PRODUCTION_MODE
    };

} // namespace b
]==])
file(WRITE ${EMBED_BINARY_DIR}/embed_source_file_template.cpp "${EMBED_SOURCE_FILE_TEMPLATE}")

# The single b::embed master source file which is compiled into the app
set(EMBED_MASTER_SOURCE_FILE [==[
// File generated using battery::embed (https://github.com/batterycenter/embed)
// DO NOT EDIT THIS FILE!!!

#include "battery/embed.hpp"

#if !defined(B_PRODUCTION_MODE) && !defined(B_OS_WEB)
#include <codecvt>
#include <thread>
#include <filesystem>
#include <fstream>
#include <mutex>
#include <unordered_map>
#include <optional>
#include <atomic>
#endif // !B_PRODUCTION_MODE and !B_OS_WEB

namespace b {
    
#if !defined(B_PRODUCTION_MODE) && !defined(B_OS_WEB)

    // All this crazyness is necessary because we need to use std::string_view because of constexpr,
    // and the string_view only points to a char array, so we need a string that outlives 
    // the string_view it is assigned to.

    struct FileData {
        std::string filename;
        std::string newContent;
        std::string filepath;
        std::filesystem::file_time_type lastWriteTime;
        std::function<void(const b::EmbedInternal::EmbeddedFile&)> callback;
    };

    static std::mutex embeddedFilesMapMutex;
    static std::unordered_map<std::string,FileData> embeddedFiles;
   
#ifdef _WIN32
#ifdef __clang__
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wdeprecated-declarations"
#endif

    static std::wstring embed_widen(const std::string& str) {
        std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
        return converter.from_bytes(str);
    }
    
#ifdef __clang__
#pragma GCC diagnostic pop
#endif

#endif // _WIN32

    static std::optional<std::string> embed_read_file(const std::string_view& filename) {
#ifdef _WIN32
        std::ifstream file(embed_widen(std::string(filename)).c_str(), std::ios::binary);
#else // _WIN32
        std::ifstream file(std::string(filename), std::ios::binary);
#endif // _WIN32
        if (file.fail()) {
            return {};
        }
        return std::string((std::istreambuf_iterator<char>(file)), std::istreambuf_iterator<char>());
    }

    // Apple-Clang does not support std::jthread yet, hence we make our own version of jthread here:
    class AutoThread {
    public:
        template<typename Tfunc>
        AutoThread(Tfunc&& function)
            : m_thread([&]() { function(m_shouldStop); }) 
        {}
        
        ~AutoThread() {
            m_shouldStop = true;
            m_thread.join();
        }

        AutoThread(const AutoThread& other) = delete;
        AutoThread(AutoThread&& other) = delete;
        AutoThread& operator=(const AutoThread& other) = delete;
        AutoThread& operator=(AutoThread&& other) = delete;

    private:
        std::thread m_thread;
        std::atomic<bool> m_shouldStop = false;
    };

    // This is a global file watcher thread that is created at program start and automatically destroyed at program end
    // All embedded files are hot-reloaded by this thread.
    AutoThread embeddedFileWatcherThread = AutoThread([](std::atomic<bool>& shouldStop) {
        while (!shouldStop) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::lock_guard<std::mutex> lock(embeddedFilesMapMutex);
            for (auto& [key, filedata] : embeddedFiles) {
                std::filesystem::path filepath(key);
                std::error_code ec;
                auto newWriteTime = std::filesystem::last_write_time(filepath, ec);
                if (!ec) {
                    if (newWriteTime != filedata.lastWriteTime) {
                        filedata.lastWriteTime = newWriteTime;
                        auto fileresult = embed_read_file(key);
                        if (!fileresult) {
                            continue;
                        }
                        filedata.newContent = *fileresult;
                        EmbedInternal::EmbeddedFile newFile(filedata.newContent.c_str(), filedata.newContent.size(), filedata.filename.c_str(), filedata.filepath.c_str());
                        filedata.callback(newFile);
                    }
                }
            }
        }
    });

#endif // !B_PRODUCTION_MODE and !B_OS_WEB

    void EmbedInternal::EmbeddedFile::get(const std::function<void(const b::EmbedInternal::EmbeddedFile&)>& callback) {
#if defined(B_PRODUCTION_MODE) || defined(B_OS_WEB)
        callback(*this);
#else // B_PRODUCTION_MODE or B_OS_WEB
        auto fileresult = embed_read_file(m_fullFilepath);
        if (fileresult) {
            m_storage = std::move(*fileresult);
            m_data = reinterpret_cast<const unsigned char*>(m_storage.c_str());
            m_size = m_storage.size();
        }
        callback(*this);

        FileData fileData;
        fileData.filename = m_filename;
        fileData.newContent = m_data;
        fileData.filepath = m_fullFilepath;
        fileData.callback = callback;
        fileData.lastWriteTime = std::filesystem::last_write_time(m_fullFilepath);

        std::lock_guard<std::mutex> lock(embeddedFilesMapMutex);
        embeddedFiles[std::string(m_fullFilepath)] = fileData;
#endif // B_PRODUCTION_MODE or B_OS_WEB
    }

} // namespace b
]==])
file(WRITE ${EMBED_BINARY_DIR}/embed_impl.cpp "${EMBED_MASTER_SOURCE_FILE}")

# The common battery::embed header and source files
set(EMBED_HEADER_FILE [=[
// File generated by battery::embed
// DO NOT EDIT THIS FILE!!!
#ifndef BATTERY_EMBED_HPP
#define BATTERY_EMBED_HPP
#include <vector>
#include <string>
#include <stdexcept>
#include <sstream>
#include <functional>
#include <cstdint>
#include <algorithm>

namespace b {
    struct EmbedInternal {
        class EmbeddedFile {
        public:
            EmbeddedFile() = default;

            EmbeddedFile(
                const unsigned char* data, size_t size,
                const char* filename
#ifndef B_PRODUCTION_MODE
                , const char* fullFilepath
#endif
            )
                : m_data(data)
                , m_size(size)
                , m_filename(filename)
#ifndef B_PRODUCTION_MODE
                , m_fullFilepath(fullFilepath)
#endif
            {}

            [[nodiscard]] std::string str() const {
                return std::string(reinterpret_cast<const char*>(m_data), m_size);
            }
            [[nodiscard]] const unsigned char* data() const { return m_data; }
            [[nodiscard]] std::vector<uint8_t> vec() const {
                return { m_data, m_data + m_size };
            }
            [[nodiscard]] size_t size() const { return m_size; }
            [[nodiscard]] size_t length() const { return m_size; }

            operator std::string() const { return str(); }
            operator std::vector<uint8_t>() const { return vec(); }

            void get(const std::function<void(const b::EmbedInternal::EmbeddedFile&)>& callback);

        private:
            const unsigned char* m_data = nullptr;
            size_t m_size = 0;
            const char* m_filename = nullptr;
#ifndef B_PRODUCTION_MODE
            std::string m_storage;
            const char* m_fullFilepath = nullptr;
#endif
        };

        ${EMBEDDED_FILES_DECLARATIONS}
    };
    
    template<class S>
    EmbedInternal::EmbeddedFile embed(const S& identifier) {
        ${EMBEDDED_FILES_RETURNS}
        throw std::runtime_error("[b::embed<>] No such file or directory");
    }

    // Функция для получения списка всех встроенных файлов
    [[nodiscard]] inline std::vector<std::string> embed_list() {
        return {
            ${EMBEDDED_FILES_LIST}
        };
    }

} // namespace b

inline std::ostream& operator<<(std::ostream& stream, const b::EmbedInternal::EmbeddedFile& file) {
    stream << file.str();
    return stream;
}

#endif // BATTERY_EMBED_HPP
]=])
file(WRITE ${EMBED_BINARY_DIR}/embed_header_file_template.hpp "${EMBED_HEADER_FILE}")

# The cmake script to embed the files. This is called later on-demand, in a separate process
set(EMBED_GENERATE_SCRIPT [=[
file(READ "${FULL_PATH}" HEX_DATA HEX)
string(LENGTH "${HEX_DATA}" HEX_LENGTH)
math(EXPR BYTE_COUNT "${HEX_LENGTH} / 2")

# Преобразуем hex-строку: 48656c6c6f → 0x48, 0x65, 0x6c, 0x6c, 0x6f
set(GENERATED_BYTE_ARRAY "${HEX_DATA}")

# Шаг 1: добавляем 0x перед каждой парой
string(REGEX REPLACE "([0-9a-f][0-9a-f])" "0x\\1" GENERATED_BYTE_ARRAY "${GENERATED_BYTE_ARRAY}")

# Шаг 2: добавляем запятую и пробел после каждого байта
string(REGEX REPLACE "(0x[0-9a-f][0-9a-f])" "\\1, " GENERATED_BYTE_ARRAY "${GENERATED_BYTE_ARRAY}")

# Шаг 3: удаляем лишнюю запятую и пробел в конце
string(REGEX REPLACE ", $" "" GENERATED_BYTE_ARRAY "${GENERATED_BYTE_ARRAY}")

# Устанавливаем размер
math(EXPR FILESIZE "${BYTE_COUNT}")

configure_file(${INFILE} ${OUTFILE})
]=])

file(WRITE ${EMBED_BINARY_DIR}/generate.cmake ${EMBED_GENERATE_SCRIPT})

set(EMBED_IDENTIFIERS "" CACHE INTERNAL "list of all identifiers used by battery::embed")
set(EMBED_FILENAMES "" CACHE INTERNAL "list of all filenames used by battery::embed")
set(EMBED_TARGETS "" CACHE INTERNAL "list of all targets used by battery::embed")

# Defer the function call until the end of the configure step
cmake_language(DEFER DIRECTORY ${CMAKE_SOURCE_DIR} CALL _embed_generate_all_hpps())
function(_embed_generate_all_hpps)
    message(STATUS "Generating b::embed() HPP files ...")
    foreach (TARGET ${EMBED_TARGETS})
        _embed_generate_hpp(${TARGET})
    endforeach()
    message(STATUS "Generating b::embed() HPP files ... Done")
endfunction(_embed_generate_all_hpps)

function(_embed_generate_hpp TARGET)
    string(TOLOWER "${TARGET}" TARGET)
    string(REGEX REPLACE "[^a-zA-Z0-9_]" "_" TARGET "${TARGET}")
    if (NOT EMBED_IDENTIFIERS)
        return()
    endif()

    set(EMBEDDED_FILES_DECLARATIONS "")
    set(EMBEDDED_FILES_RETURNS "")
    set(EMBEDDED_FILES_LIST "")

    # Iterate over indices
    list(LENGTH EMBED_IDENTIFIERS num_identifiers)
    math(EXPR last_index "${num_identifiers} - 1")
    foreach(INDEX RANGE 0 ${last_index})
        list(GET EMBED_IDENTIFIERS ${INDEX} IDENTIFIER)
        list(GET EMBED_FILENAMES ${INDEX} FILENAME)
        if (IDENTIFIER MATCHES "^${TARGET}_")
            set(EMBEDDED_FILES_DECLARATIONS "${EMBEDDED_FILES_DECLARATIONS}static EmbeddedFile ${IDENTIFIER};\n")

            set(EMBEDDED_FILES_RETURNS "${EMBEDDED_FILES_RETURNS}if (identifier == \"${FILENAME}\") { return EmbedInternal::${IDENTIFIER}; }\nelse ")

            set(EMBEDDED_FILES_LIST "${EMBEDDED_FILES_LIST}    \"${FILENAME}\",\n")
        endif()
    endforeach()

    # Remove the last comma
    string(REGEX REPLACE ",\n$" "" EMBEDDED_FILES_LIST "${EMBEDDED_FILES_LIST}")

    # Generate the header file
    file(READ ${EMBED_BINARY_DIR}/embed_header_file_template.hpp EMBED_HEADER_FILE)
    string(CONFIGURE "${EMBED_HEADER_FILE}" EMBED_HEADER_FILE_GENERATED)

    # Path to embed.hpp
    set(EMBED_HPP "${EMBED_BINARY_DIR}/autogen/${TARGET}/include/battery/embed.hpp")
    get_filename_component(EMBED_HPP_DIR "${EMBED_HPP}" DIRECTORY)
    file(MAKE_DIRECTORY "${EMBED_HPP_DIR}")
    file(WRITE "${EMBED_HPP}" "${EMBED_HEADER_FILE_GENERATED}")
endfunction()

# Internal function for validating an identifier
function(embed_validate_identifier IDENTIFIER)  # Validate the identifier against C variable naming rules
    if (NOT IDENTIFIER MATCHES "^[a-zA-Z_][a-zA-Z0-9_]*$")
        message(FATAL_ERROR "embed: Identifier contains invalid characters: '${IDENTIFIER}'")
    endif()
endfunction()

# The main function for embedding files
function(b_embed TARGET FILENAME)
    string(TOLOWER "${TARGET}" TARGET_ID)
    string(REGEX REPLACE "[^a-zA-Z0-9_]" "_" TARGET_ID "${TARGET_ID}")
    if (IS_ABSOLUTE "${FILENAME}")
        message(FATAL_ERROR "embed: File name must be relative to the current source directory: '${FILENAME}'")
    endif()
    # Make the identifier
    string(TOLOWER "${TARGET_ID}_${FILENAME}" IDENTIFIER)
    string(REGEX REPLACE "[^a-zA-Z0-9_]" "_" IDENTIFIER "${IDENTIFIER}") # Replace all invalid characters with underscores
    embed_validate_identifier("${IDENTIFIER}") # Validate the identifier against C variable naming rules
    # Set up paths
    get_filename_component(FULL_PATH "${FILENAME}" ABSOLUTE) # Make the file path absolute
    set(CPP_FILE "${EMBED_BINARY_DIR}/autogen/${TARGET_ID}/src/${IDENTIFIER}.cpp")
    # If identifier already in use
    list(FIND EMBED_IDENTIFIERS ${IDENTIFIER} EMBED_USED_IDENTIFIERS_INDEX)
    if (NOT EMBED_USED_IDENTIFIERS_INDEX EQUAL -1)
        message(FATAL_ERROR "embed: Identifier already in use: '${IDENTIFIER}'")
    endif()
    # === Add identifier and file name ===
    set(EMBED_IDENTIFIERS ${EMBED_IDENTIFIERS} ${IDENTIFIER} CACHE INTERNAL "list of all identifiers used by the embed library" FORCE)
    set(EMBED_FILENAMES ${EMBED_FILENAMES} ${FILENAME} CACHE INTERNAL "list of all filenames used by the embed library" FORCE)
    set(EMBED_TARGETS ${EMBED_TARGETS} ${TARGET} CACHE INTERNAL "list of all targets used by the embed library" FORCE)
    
    # === Generate embed.hpp lazy ===
    _embed_generate_hpp(${TARGET})
    
    # === Generate .cpp file for embed ===
    set(EMBED_HPP "${EMBED_BINARY_DIR}/autogen/${TARGET_ID}/include/battery/embed.hpp")
    set(EMBED_CPP_TEMPLATE "${EMBED_BINARY_DIR}/embed_source_file_template.cpp")
    add_custom_command(
        COMMAND ${CMAKE_COMMAND}
            -DINFILE=${EMBED_CPP_TEMPLATE}
            -DOUTFILE=${CPP_FILE}
            -DFULL_PATH=${FULL_PATH}
            -DIDENTIFIER=${IDENTIFIER}
            -DFILENAME=${FILENAME}
            -P "${EMBED_BINARY_DIR}/generate.cmake"
        DEPENDS "${FULL_PATH}" "${EMBED_HPP}" "${EMBED_BINARY_DIR}/generate.cmake" "${EMBED_BINARY_DIR}/embed_source_file_template.cpp"
        OUTPUT ${CPP_FILE}
        VERBATIM
    )

    # === Adding files to the target (embed.hpp now already exists!) ===
    target_include_directories(${TARGET} PUBLIC 
        $<BUILD_INTERFACE:${EMBED_BINARY_DIR}/autogen/${TARGET_ID}/include>
        $<INSTALL_INTERFACE:>
    )
    target_sources(${TARGET} PRIVATE ${CPP_FILE} ${EMBED_BINARY_DIR}/embed_impl.cpp)
    target_compile_definitions(${TARGET} PRIVATE _SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING)

    if (NOT B_PRODUCTION_MODE)
        find_package(Threads REQUIRED)
        target_link_libraries(${TARGET} PRIVATE Threads::Threads)
    else ()
        target_compile_definitions(${TARGET} PUBLIC B_PRODUCTION_MODE)
    endif ()

    if (MSVC)
        target_sources(${TARGET} PRIVATE ${FULL_PATH})
        source_group(TREE ${EMBED_BINARY_DIR}/autogen/${TARGET_ID}/src PREFIX "embed/autogen" FILES ${CPP_FILE})
        source_group(TREE ${EMBED_BINARY_DIR} PREFIX "embed/autogen" FILES ${EMBED_BINARY_DIR}/embed_impl.cpp)
        source_group(TREE ${EMBED_BINARY_DIR}/autogen/${TARGET_ID}/include/battery PREFIX "embed/autogen" FILES ${EMBED_HPP})
        source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" PREFIX "/embed" FILES ${FULL_PATH})
    endif()
endfunction()

function(b_embed_proxy_target MAIN_TARGET PROXY_TARGET)
    add_library(${PROXY_TARGET} STATIC)
    target_compile_features(${PROXY_TARGET} PUBLIC cxx_std_20)
    target_link_libraries(${MAIN_TARGET} PRIVATE ${PROXY_TARGET})
    set_target_properties(${PROXY_TARGET} PROPERTIES FOLDER "embed-proxy-targets")
endfunction()

# Build all examples
if (B_EMBED_BUILD_EXAMPLES)
    add_subdirectory(examples)      # Set Example 'simple' as the default project in Visual Studio
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT simple)
endif()

# Set the predefined targets folder for Visual Studio
if (PROJECT_IS_TOP_LEVEL)
    set(PREDEFINED_TARGETS_FOLDER "CMakePredefinedTargets")
endif()

if (NOT B_EMBED_SILENCE_DEVMODE_WARNING AND NOT B_PRODUCTION_MODE)
    message(WARNING "Battery::Embed is in development mode, enabling hotreload. Before deploying your application, use -DB_PRODUCTION_MODE=ON to strip all development features such as hotreload and absolute filepaths.")
endif ()
