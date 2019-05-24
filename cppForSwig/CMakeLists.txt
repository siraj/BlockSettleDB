set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

add_subdirectory(chacha20poly1305)
add_subdirectory(lmdb/libraries/liblmdb)

# this is the right way to set options for sub-projects
# we allow these to be overridden by the user
if(NOT DEFINED LIBBTC_WITH_WALLET)
    set(LIBBTC_WITH_WALLET OFF CACHE BOOL "disable libbtc wallet"     FORCE)
endif()

if(NOT DEFINED LIBBTC_WITH_NET)
    set(LIBBTC_WITH_NET    OFF CACHE BOOL "disable libbtc networking" FORCE)
endif()

if(NOT DEFINED LIBBTC_WITH_TOOLS)
    set(LIBBTC_WITH_TOOLS  OFF CACHE BOOL "disable libbtc tools"      FORCE)
endif()

if(NOT DEFINED LIBBTC_WITH_TESTS)
    set(LIBBTC_WITH_TESTS  OFF CACHE BOOL "do not build libbtc tests" FORCE)
endif()

add_subdirectory(libbtc)

set(LIBARMORYCOMMON_INCLUDE_DIRECTORIES
    PUBLIC lmdb/libraries/liblmdb
    PUBLIC .
    PRIVATE bech32/ref/c++
    PRIVATE chacha20poly1305
    PRIVATE libbtc/src/secp256k1/include
    PRIVATE libbtc/include
    PRIVATE ${PROJECT_BINARY_DIR} # for protobuf protoc genned .h files
)

set(LIBARMORYCOMMON_LINK_LIBRARIES
    pthread
    chacha20poly1305
    btc
    lmdb
)

option(WITH_CRYPTOPP "use Crypto++ crypto functions" OFF)

unset(LIBARMORYCOMMON_SOURCES)
unset(LIBARMORYCOMMON_COMPILE_DEFINITIONS)

if(WITH_CRYPTOPP)
    add_subdirectory(cryptopp)

    list(APPEND LIBARMORYCOMMON_SOURCES EncryptionUtils.cpp)

    list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
        cryptopp
    )
else()
    list(APPEND LIBARMORYCOMMON_COMPILE_DEFINITIONS
        PRIVATE -DLIBBTC_ONLY
    )

    list(APPEND LIBARMORYCOMMON_SOURCES EncryptionUtils_libbtc.cpp)
endif()

find_package(PkgConfig REQUIRED)

pkg_check_modules(LIBWEBSOCKETS REQUIRED libwebsockets>=3.0.1)

list(APPEND LIBARMORYCOMMON_INCLUDE_DIRECTORIES
    PRIVATE ${LIBWEBSOCKETS_INCLUDE_DIRS}
)

list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
    ${LIBWEBSOCKETS_LIBRARIES}
)

include(CheckCXXSymbolExists)

check_cxx_symbol_exists(LWS_WITH_TLS lws_config.h LIBWEBSOCKETS_WITH_TLS)

if(LIBWEBSOCKETS_WITH_TLS)
    if(APPLE)
        set(OPENSSL_ROOT_DIR /usr/local/opt/openssl)
    endif()

    find_package(OpenSSL REQUIRED)

    list(APPEND LIBARMORYCOMMON_INCLUDE_DIRECTORIES
        PUBLIC ${OPENSSL_INCLUDE_DIR}
    )

    list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
        ${OPENSSL_LIBRARIES}
    )
endif()

check_cxx_symbol_exists(LWS_WITH_LIBUV lws_config.h LIBWEBSOCKETS_WITH_LIBUV)

if(LIBWEBSOCKETS_WITH_LIBUV)
    pkg_check_modules(LIBUV REQUIRED libuv)

    list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
        ${LIBUV_LIBRARIES}
    )
endif()

check_cxx_symbol_exists(LWS_WITH_LIBEVENT lws_config.h LIBWEBSOCKETS_WITH_LIBEVENT)

if(LIBWEBSOCKETS_WITH_LIBEVENT)
    pkg_check_modules(LIBEVENT REQUIRED libevent)

    list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
        ${LIBEVENT_LIBRARIES}
    )
endif()

check_cxx_symbol_exists(LWS_WITH_LIBCAP lws_config.h LIBWEBSOCKETS_WITH_LIBCAP)

if(LIBWEBSOCKETS_WITH_LIBCAP)
    pkg_check_modules(LIBCAP REQUIRED libcap)

    list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
        ${LIBCAP_LIBRARIES}
    )
endif()

pkg_check_modules(PROTOBUF REQUIRED protobuf>=2.0.0)

list(APPEND LIBARMORYCOMMON_INCLUDE_DIRECTORIES
    PRIVATE ${PROTOBUF_INCLUDE_DIRS}
)

list(APPEND LIBARMORYCOMMON_LINK_LIBRARIES
    ${PROTOBUF_LIBRARIES}
)

list(APPEND LIBARMORYCOMMON_SOURCES
    Accounts.cpp
    Addresses.cpp
    AssetEncryption.cpp
    Assets.cpp
    AsyncClient.cpp
    AuthorizedPeers.cpp
    BinaryData.cpp
    BIP32_Node.cpp
    BlockDataManagerConfig.cpp
    BtcUtils.cpp
    ClientClasses.cpp
    CoinSelection.cpp
    DecryptedDataContainer.cpp
    DerivationScheme.cpp
    hkdf.cpp
    KDF.cpp
    log.cpp
    NetworkConfig.cpp
    ReentrantLock.cpp
    Script.cpp
    SecureBinaryData.cpp
    ScriptRecipient.cpp
    Signer.cpp
    SocketObject.cpp
    StoredBlockObj.cpp
    SwigClient.cpp
    Transactions.cpp
    TxClasses.cpp
    TxEvalState.cpp
    txio.cpp
    UniversalTimer.cpp
    WalletManager.cpp
    Wallets.cpp
    WebSocketClient.cpp
    WebSocketMessage.cpp
    DBUtils.cpp
    bech32/ref/c++/bech32.cpp
    bech32/ref/c++/segwit_addr.cpp
)

set(PROTOBUF_FILES
    AddressBook.proto
    AddressData.proto
    BDVCommand.proto
    CommonTypes.proto
    FeeEstimate.proto
    LedgerEntry.proto
    NodeStatus.proto
    Utxo.proto
)

find_program(PROTOC_EXECUTABLE protoc)

if(NOT PROTOC_EXECUTABLE)
    message(FATAL_ERROR "protocol buffers compiler 'protoc' not found")
endif()

file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/protobuf)

foreach(proto ${PROTOBUF_FILES})
    string(REGEX REPLACE "\\.proto$" ".pb.cc" proto_cc ${proto})

    add_custom_command(
        OUTPUT ${PROJECT_BINARY_DIR}/protobuf/${proto_cc}
        COMMAND ${PROTOC_EXECUTABLE} --cpp_out=${PROJECT_BINARY_DIR}/protobuf --proto_path=${CMAKE_CURRENT_SOURCE_DIR}/protobuf ${CMAKE_CURRENT_SOURCE_DIR}/protobuf/${proto}
        DEPENDS protobuf/${proto}
    )

    list(APPEND LIBARMORYCOMMON_SOURCES ${PROJECT_BINARY_DIR}/protobuf/${proto_cc})
endforeach()

add_library(ArmoryCommon
    STATIC
    ${LIBARMORYCOMMON_SOURCES}
)

target_compile_definitions(ArmoryCommon
    PUBLIC -D__STDC_LIMIT_MACROS
    ${LIBARMORYCOMMON_COMPILE_DEFINITIONS}
)

target_include_directories(ArmoryCommon
    ${LIBARMORYCOMMON_INCLUDE_DIRECTORIES}
)

target_link_libraries(ArmoryCommon
    ${LIBARMORYCOMMON_LINK_LIBRARIES}
)

set(LIBARMORYCLI_SOURCES
    BDM_mainthread.cpp
    BDM_Server.cpp
    BIP150_151.cpp
    BitcoinP2P.cpp
    Blockchain.cpp
    BlockchainScanner.cpp
    BlockchainScanner_Super.cpp
    BlockDataMap.cpp
    BlockDataViewer.cpp
    BlockObj.cpp
    BlockUtils.cpp
    BtcWallet.cpp
    DatabaseBuilder.cpp
    HistoryPager.cpp
    HttpMessage.cpp
    JSON_codec.cpp
    LedgerEntry.cpp
    lmdb_wrapper.cpp
    nodeRPC.cpp
    Progress.cpp
    ScrAddrFilter.cpp
    ScrAddrObj.cpp
    Server.cpp
    SocketService_unix.cpp
    SshParser.cpp
    StringSockets.cpp
    txio.cpp
    ZeroConf.cpp
)

add_library(ArmoryCLI
    STATIC
    ${LIBARMORYCLI_SOURCES}
)

target_link_libraries(ArmoryCLI
    ArmoryCommon
)

target_compile_definitions(ArmoryCLI
    PUBLIC -D__STDC_LIMIT_MACROS
)

string_option(WITH_GUI "with GUI" AUTO)

find_package(Qt4 COMPONENTS QtGui)

find_program(PYRCC4_EXECUTABLE   pyrcc4)
find_program(LRELEASE_EXECUTABLE lrelease)

if(WITH_GUI STREQUAL AUTO AND Qt4_FOUND AND PYRCC4_EXECUTABLE AND LRELEASE_EXECUTABLE)
    set(WITH_GUI ON)
endif()

if(WITH_GUI)
    if(NOT PYRCC4_EXECUTABLE)
        message(FATAL_ERROR "pyrcc4 not found, install python2-qt4-devel")
    endif()

    if(NOT LRELEASE_EXECUTABLE)
        message(FATAL_ERROR "lrelease not found, install qt4-linguist")
    endif()

    add_library(ArmoryGUI
        TransactionBatch.cpp
    )

    target_link_libraries(ArmoryGUI
        ArmoryCommon
        Qt4::QtGui
    )

    target_compile_definitions(ArmoryGUI
        PUBLIC -D__STDC_LIMIT_MACROS
    )
endif()

add_executable(ArmoryDB
    main.cpp
)

set_target_properties(ArmoryDB
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
)

target_link_libraries(ArmoryDB
    ArmoryCLI
)

add_executable(BIP150KeyManager
    KeyManager.cpp
)

set_target_properties(BIP150KeyManager
    PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
)

target_link_libraries(BIP150KeyManager
    ArmoryCLI
)

string_option(WITH_CLIENT "build python client" AUTO)

find_package(Python2 COMPONENTS Development)

find_program(SWIG_EXECUTABLE swig)

if(WITH_CLIENT STREQUAL AUTO AND Python2_Development_FOUND AND SWIG_EXECUTABLE)
    set(WITH_CLIENT ON)
endif()

if(WITH_CLIENT)
    set(CPPBLOCKUTILS_SOURCES
        ${PROJECT_BINARY_DIR}/CppBlockUtils_wrap.cxx
    )

    unset(SWIG_OPTS)

    if(CMAKE_COMPILER_IS_GNUCXX)
        set(SWIG_OPTS -D__GNUC__)
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL Clang)
        set(SWIG_OPTS -D__CLANG__)
    endif()

    if(CMAKE_C_SIZEOF_DATA_PTR EQUAL 8) # 64 bit
        set(SWIG_OPTS ${SWIG_OPTS} -D__LP64__)
    endif()

    add_custom_command(
        OUTPUT ${PROJECT_BINARY_DIR}/CppBlockUtils_wrap.cxx
        COMMAND ${SWIG_EXECUTABLE} -c++ -python -threads ${SWIG_OPTS} -o ${PROJECT_BINARY_DIR}/CppBlockUtils_wrap.cxx ${CMAKE_CURRENT_SOURCE_DIR}/CppBlockUtils.i
        DEPENDS CppBlockUtils.i
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
    )

    set(CPPBLOCKUTILS_LINK_LIBRARIES
        ArmoryCommon
        Python2::Python
    )

    add_library(CppBlockUtils
        SHARED
        ${CPPBLOCKUTILS_SOURCES}
    )

    set_target_properties(CppBlockUtils
        PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
    )

    target_include_directories(CppBlockUtils
        PUBLIC Python2::Python
    )

    if(WITH_GUI)
        list(APPEND CPPBLOCKUTILS_LINK_LIBRARIES
            ArmoryGUI
        )

        # this mechanism for dependent custom targets I found here:
        # https://cmake.org/pipermail/cmake/2010-March/035621.html
        add_custom_command(
            OUTPUT  ${PROJECT_BINARY_DIR}/qrc_img_resources.py
            COMMAND ${PYRCC4_EXECUTABLE} -o ${PROJECT_BINARY_DIR}/qrc_img_resources.py ${PROJECT_SOURCE_DIR}/imgList.xml
            DEPENDS ${PROJECT_SOURCE_DIR}/imgList.xml
            WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
        )

        add_custom_target(qrc_img_resources_py DEPENDS ${PROJECT_BINARY_DIR}/qrc_img_resources.py)

        file(COPY ${PROJECT_SOURCE_DIR}/lang DESTINATION ${PROJECT_BINARY_DIR})

        set(LANG_SOURCES
            armory_da.ts
            armory_de.ts
            armory_el.ts
            armory_en.ts
            armory_es.ts
            armory_fr.ts
            armory_he.ts
            armory_hr.ts
            armory_id.ts
            armory_ru.ts
            armory_sv.ts
        )

        unset(LANG_QM_FILES)

        foreach(lang_file ${LANG_SOURCES})
            string(REGEX REPLACE "\\.ts$" ".qm" lang_qm_file ${lang_file})

            add_custom_command(
                OUTPUT  ${PROJECT_BINARY_DIR}/lang/${lang_qm_file}
                COMMAND ${LRELEASE_EXECUTABLE} ${PROJECT_BINARY_DIR}/lang/${lang_file}
                WORKING_DIRECTORY ${PROJECT_BINARY_DIR}/lang
            )

            list(APPEND LANG_QM_FILES
                ${PROJECT_BINARY_DIR}/lang/${lang_qm_file}
            )
        endforeach()

        add_custom_target(lang_lrelease DEPENDS ${LANG_QM_FILES})

        add_dependencies(CppBlockUtils
            qrc_img_resources_py
            lang_lrelease
        )
    endif()

    target_link_libraries(CppBlockUtils
        ${CPPBLOCKUTILS_LINK_LIBRARIES}
    )
endif()

option(ENABLE_TESTS "build tests binaries" OFF)

if(ENABLE_TESTS)
    add_subdirectory(gtest)
endif()