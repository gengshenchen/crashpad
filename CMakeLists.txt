# 3rdparty/crashpad/CMakeLists.txt

# 設置一個最小的 CMake 版本要求
cmake_minimum_required(VERSION 3.15)

# -----------------------------------------------------------------------------
# 1. 偵測平台並設定函式庫路徑
#    讓這份設定檔可以跨平台使用 (Linux, Windows, macOS)
# -----------------------------------------------------------------------------
if(WIN32)
    set(CRASHPAD_LIB_SUBDIR "lib/win")
    # 在 Windows 上，你可能還需要區分 x86 和 x64
    # if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    #     set(CRASHPAD_LIB_SUBDIR "lib/win/x64")
    # else()
    #     set(CRASHPAD_LIB_SUBDIR "lib/win/x86")
    # endif()
elseif(APPLE)
    set(CRASHPAD_LIB_SUBDIR "lib/mac")
elseif(UNIX AND NOT APPLE)
    set(CRASHPAD_LIB_SUBDIR "lib/linux")
else()
    message(FATAL_ERROR "Crashpad: Unsupported platform")
endif()

set(CRASHPAD_LIB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${CRASHPAD_LIB_SUBDIR})
message(STATUS "Configuring Crashpad for ${CMAKE_SYSTEM_NAME}, using libs from: ${CRASHPAD_LIB_DIR}")

# -----------------------------------------------------------------------------
# 2. 定義所有 IMPORTED 函式庫
#    我們使用 crashpad:: 作為命名空間，避免與其他函式庫名稱衝突
# -----------------------------------------------------------------------------

# --- mini_chromium_base ---
# 這是最底層的依賴，沒有其他 INTERFACE_LINK_LIBRARIES
add_library(crashpad::base STATIC IMPORTED)
set_target_properties(crashpad::base PROPERTIES
    IMPORTED_LOCATION "${CRASHPAD_LIB_DIR}/libbase.a"
)

# --- crashpad_util ---
add_library(crashpad::util STATIC IMPORTED)
set_target_properties(crashpad::util PROPERTIES
    IMPORTED_LOCATION "${CRASHPAD_LIB_DIR}/libutil.a"
    INTERFACE_LINK_LIBRARIES "crashpad::base" # util 依賴 base
)

# --- crashpad_common ---
add_library(crashpad::common STATIC IMPORTED)
set_target_properties(crashpad::common PROPERTIES
    IMPORTED_LOCATION "${CRASHPAD_LIB_DIR}/libcommon.a"
    INTERFACE_LINK_LIBRARIES "crashpad::util" # common 依賴 util
)

# --- crashpad_client ---
add_library(crashpad::client STATIC IMPORTED)
set_target_properties(crashpad::client PROPERTIES
    IMPORTED_LOCATION "${CRASHPAD_LIB_DIR}/libclient.a"
    INTERFACE_LINK_LIBRARIES "crashpad::util" # client 也依賴 util
)

# -----------------------------------------------------------------------------
# 3. 設定頭文件路徑
#    這些路徑會透過 INTERFACE 屬性被傳遞給任何連結到這些目標的對象
# -----------------------------------------------------------------------------
set(CRASHPAD_INCLUDE_DIRS
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/third_party/mini_chromium"
)

# 為所有函式庫設定 INTERFACE include 路徑
target_include_directories(crashpad::base INTERFACE ${CRASHPAD_INCLUDE_DIRS})
target_include_directories(crashpad::util INTERFACE ${CRASHPAD_INCLUDE_DIRS})
target_include_directories(crashpad::common INTERFACE ${CRASHPAD_INCLUDE_DIRS})
target_include_directories(crashpad::client INTERFACE ${CRASHPAD_INCLUDE_DIRS})

# -----------------------------------------------------------------------------
# 4. 建立一個「接口庫 (Interface Library)」來封裝所有目標
#    這就是我們最終要給主項目使用的目標，類似 spdlog::spdlog
# -----------------------------------------------------------------------------
add_library(crashpad_interface INTERFACE)
target_link_libraries(crashpad_interface INTERFACE
    crashpad::client   # 連結 client
    crashpad::common   # 連結 common
    # crashpad::util 和 crashpad::base 會被 CMake 作為依賴自動帶入，所以這裡可以省略
)

# 如果有全域的系統依賴，也可以加在這裡
# 例如在 Linux 上，Crashpad 可能需要 pthread
if(UNIX)
    # target_link_libraries(crashpad::crashpad INTERFACE Threads::Threads)
endif()

# 為這個目標建立一個帶有命名空間的 ALIAS (別名)
#    主項目將會使用這個別名進行連結
add_library(crashpad::crashpad ALIAS crashpad_interface)

# 提示：你已經成功建立了一個名為 crashpad::crashpad 的目標
message(STATUS "Created target 'crashpad::crashpad' for easy integration.")
