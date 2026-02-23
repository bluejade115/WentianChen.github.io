# CMakeLists.txt
## 基础概念
### Package Configuration File
包配置文件通常命名为`YourPackageConfig.cmake` 或 `yourpackage-config.cmake`。它们由包的作者提供，包含了关于如何使用该包的所有必要信息，例如头文件路径、库文件路径以及它所依赖的其他包。或者名为 `FindYourPackage.cmake` 的文件。这些文件通常由用户或第三方编写，用于查找那些没有提供标准包配置文件的传统库。它们包含了一些启发式逻辑来定位库和头文件。

包配置文件的存储位置:如果使用安装类型的编译（即编译完一个包后，将相关文件安装到`your_workspace/install/your_package`），那么`*.cmake`通常会在`share/your_package/cmake` ，其中`your_workspace/install/your_package`称之为**包的路径前缀**(prefix)。

### setup.bash的作用
当终端启动后，执行工作空间的`setup.bash`后，会将工作空间内的**包的路径前缀**添加到变量`CMAKE_PREFIX_PATH`里。

确认包是否正确被添加的排查方法如下：

1. 确认包的路径前缀在变量`CMAKE_PREFIX_PATH`里，通常正常编译后都会存在。
2. 确认包存在于`your_workspace/install/your_package`,确定安装到`install`目录下。
3. 确认存在包配置文件，检查`share/your_package/cmake`里是否存在相关的`.cmake`文件。

## find_package
### find_package的查找原理 
find_package 命令的查找过程相当复杂，但可以概括为两个主要模式：模块模式（Module Mode）和配置模式（Config Mode）。CMake 会按顺序尝试这两种模式来查找包。

1. 模块模式 (Module Mode)
      - 原理: CMake 自身或第三方开发者提供了`Find<PackageName>.cmake`脚本。这些脚本包含了查找特定库（如 OpenCV）所需头文件和库文件的逻辑。
      - 查找位置:
        - `CMAKE_MODULE_PATH`变量中指定的目录。
        - CMake 内置的模块目录，例如`/usr/share/cmake-<version>/Modules/`。
      - 如何确定: 如果`find_package(OpenCV)`成功，并且定义了 `OpenCV_INCLUDE_DIRS` 和 `OpenCV_LIBRARIES` 这样的变量，这说明使用了模块模式。

2. 配置模式 (Config Mode)

      - 原理: 库的开发者在安装时会生成 `<PackageName>Config.cmake` 或 `<PackageName>-config.cmake` 配置文件。这些文件包含了库的安装路径、头文件路径、库文件路径等信息。这种方式是现代 CMake 的推荐做法，因为它将配置信息与库本身一起安装，更加可靠。
      - 查找位置:
        - 在 `CMAKE_PREFIX_PATH` 变量中指定的目录。
        - 在环境变量中指定的目录，例如 `PATH`。
        - 在标准的系统路径下，如 `/usr/local/`、`/usr/` 等。
        - 在 ROS 中，catkin 会将工作区 (`devel` 或 `install` 目录) 自动添加到 `CMAKE_PREFIX_PATH`，这使得 `find_package` 能够找到 ROS 包。
      - 如何确定: 如果`find_package(Boost REQUIRED)` 成功，并且定义了 Boost::system 这样的导入目标，这意味使用了配置模式。导入目标是 CMake 推荐的现代链接方式，它将库的所有信息都封装在一个目标中。

### find_package的检索过程
当你调用 `find_package(<PackageName> REQUIRED)` 时：

- CMake 会遍历 `CMAKE_PREFIX_PATH` 中的每一个路径。
- 对于每个路径，它会尝试在以下结构中查找 `*Config.cmake` 文件： 
    - `<prefix>/share/<PackageName>/cmake/<PackageName>Config.cmake`
    - `<prefix>/lib/cmake/<PackageName>Config.cmake`
    - 等等。
- 一旦找到匹配的 `*Config.cmake` 文件，它就会加载该文件。这个文件会设置一系列变量（例如 `moveit_visual_tools_INCLUDE_DIRS`、`moveit_visual_tools_LIBRARIES`）并可能定义导入目标，从而使得你的项目能够链接到该包。

哪些包需要被添加到`find_package`: 如果你的程序使用了其他包的内容都需要写`find_package`
### 手动添加查找目录
默认的查找路径有：

- `CMAKE_PREFIX_PATH`
- `/usr/local/lib`
- `/usr/lib`
- `/usr/local/include`
- `/usr/include`

手动在`CmakeLists.txt`中添加查找路径：
```
# 设置 OpenCV 的安装路径# 将 /path/to/your/opencv/install 替换为您实际的安装路径
set(OpenCV_DIR "/path/to/your/opencv/install/lib/cmake/opencv4")  

# 或者使用 CMAKE_PREFIX_PATH，这会搜索指定目录下的 share/ 或 lib/cmake/ 子目录
set(CMAKE_PREFIX_PATH "/path/to/your/opencv/install")

# 亦或者
find_package(OpenCV REQUIRED
    HINTS "/path/to/opencv"
    "/another/path/to/search"
)
```
### 验证查找到的库
```
# 打印出找到的变量，用于验证
message(STATUS "OpenCV include directories：${OpenCV_INCLUDE_DIRS}") 
message(STATUS "OpenCV libraries: ${OpenCV_LIBRARIES}")
```
### 环境变量
`find_package`在寻找和导入包时会同时生成环境变量，比如`OpenCV_INCLUDE_DIRS`和`OpenCV_LIBRARIES`。
```
# 这一步会生成catkin相关的变量
find_package(catkin REQUIRED COMPONENTS
    roscpp
    rospy
    # ...
)

# 此时会生成：
# - catkin_INCLUDE_DIRS
# - catkin_LIBRARIES
# - catkin_LIBRARY_DIRS
# - catkin_DEPENDS
```
同时，也支持在CMakeLists.txt中手动设置环境变量
```
# catkin_INCLUDE_DIRS 包含了所有依赖包的头文件目录路径
# 例如可能包含：
# /opt/ros/noetic/include  # ROS核心头文件
# /usr/include/pcl-1.10    # PCL头文件
# /usr/include/opencv4     # OpenCV头文件

# 使用方式：
include_directories(
    ${catkin_INCLUDE_DIRS}  # 让编译器知道在哪里找头文件
)


# catkin_LIBRARIES 包含了所有依赖包的库文件列表
# 例如：
# libroscpp.so
# libroslib.so
# libopencv_core.so

# 使用方式：
target_link_libraries(你的程序名称
    ${catkin_LIBRARIES}  # 链接需要的库文件
)


# catkin_LIBRARY_DIRS 包含库文件所在的目录路径
# 例如：
# /opt/ros/noetic/lib
# /usr/local/lib

# 使用方式：
link_directories(
    ${catkin_LIBRARY_DIRS}  # 告诉链接器在哪里找库文件
)


# catkin_DEPENDS 记录了当前包依赖的其他catkin包
# 例如：roscpp rospy std_msgs

# 使用方式：
catkin_package(
    CATKIN_DEPENDS ${catkin_DEPENDS}  # 声明包的依赖关系
)
```
## add_dependencies 
`add_dependencies(...)`的作用是在构建系统中建立目标之间的依赖关系。它主要用来确保一个目标（比如一个可执行文件或库）在另一个目标之前被构建。`add_dependencies()`关注的是编译时的依赖，即在编译一个目标之前，需要先完成哪些其他目标的编译。

`add_dependencies()`主要用于以下两种情况：

1. 代码生成和编译顺序: 当你的项目需要先生成某些文件（比如消息头文件、服务头文件）然后才能编译依赖这些文件的源代码时，你需要使`add_dependencies()`。
2. 强制编译顺序: 某些情况下，即使没有直接的代码依赖，你也可能需要强制 CMake 先编译某个目标，以满足特定的构建流程要求。

你需要在以下情况中添加 `add_dependencies()`：

- ROS 消息和服务: 这是在 ROS 项目中最常见的使用场景。当你的节点使用了自定义的消息或服务文件时，catkin 需要先生成这些文件的 C++ 或 Python 头文件。你需要确保在编译你的节点之前，这些头文件已经被生成。
- 与其他目标存在非代码依赖: 如果一个目标的生成需要另一个目标的输出文件。
`add_dependencies(...)`的作用是在构建系统中建立目标之间的依赖关系。它主要用来确保一个目标（比如一个可执行文件或库）在另一个目标之前被构建。`add_dependencies()`关注的是编译时的依赖，即在编译一个目标之前，需要先完成哪些其他目标的编译。
## target_link_libraries
`target_link_libraries`的作用是链接库文件到可执行文件或库文件。它的主要目的是告诉编译器在编译你的程序时需要使用哪些已经编译好的库，以便程序可以调用这些库中定义的函数和类。`target_link_libraries()` 关注的是链接时的依赖，即程序运行时需要哪些库文件。

`target_link_libraries` 主要用于以下几个方面：

1. 链接 ROS 库: 当你的节点需要使用 ROS 提供的功能（例如发布/订阅话题、调用服务等）时，你需要链接 ROS 相关的库。例如，rospy、roscpp、message_generation 等。
2. 链接外部库: 如果你的程序使用了非 ROS 的第三方库，比如 OpenCV、PCL 等，你也需要使用 `target_link_libraries` 来链接这些库，这样你的程序才能正常调用它们提供的功能。
3. 链接自定义库: 当你的项目中包含自己编写的库（add_library 创建的），并且你想在另一个可执行文件或库中使用它时，同样需要使用 `target_link_libraries` 来进行链接。

需要在以下情况中添加 target_link_libraries：

- 创建可执行文件: 当你使用`add_executable`创建一个可执行文件时，如果这个节点需要依赖任何库，你都必须在 `target_link_libraries` 中指定这些库。
- 创建库: 当你使用 `add_library` 创建一个库时，如果这个库需要依赖其他库，你也需要使用 `target_link_libraries` 来链接它们。
### 链接ROS库
```
# 创建一个名为 "my_ros_node" 的可执行文件
add_executable(my_ros_node src/my_ros_node.cpp)

# 链接 ROS 和 catkin 相关的库
target_link_libraries(my_ros_node
  ${catkin_LIBRARIES}
)
```
`catkin_LIBRARIES` 是一个由 `catkin_make` 自动生成的变量，它包含了所有在 `package.xml` 文件中通过 `<depend>`标签声明的 ROS 依赖项。你无需手动列出 `roscpp`、`message_generation` 等库，catkin 会为你处理。你只需确保你的 `package.xml` 文件中的依赖项是正确的。
### 链接外部库
```
# 找到 OpenCV 包
find_package(OpenCV REQUIRED)

# 创建一个名为 "image_processor" 的可执行文件
add_executable(image_processor src/image_processor.cpp)

# 链接 ROS 库和 OpenCV 库
target_link_libraries(image_processor
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
)
```
在`target_link_libraries`填写库的路径。确定库的变量或者路径的方法如下：

1. 使用 `find_package()`： 首先，在 `CMakeLists.txt` 中使用 `find_package(OpenCV REQUIRED)` 来找到并加载 OpenCV 库。
2. 查看 `find_package` 文档： 几乎所有主流库的 `find_package` 模块都会定义一个或多个变量，其中通常会有一个 `_LIBRARIES` 变量，包含了链接所需的库文件路径。对于 OpenCV，这个变量就是 `OpenCV_LIBRARIES`。
3. 检查库的安装路径： 如果 `find_package` 失败，你需要确保该库已经正确安装在你的系统上。你可以在 `/usr/lib/` 或 `/usr/local/lib/` 目录下查找库文件的名称（例如 `libopencv_core.so`）。
### 链接自定义库
```
# 1. 创建自定义库
add_library(common_utils src/common_utils.cpp)

# 2. 创建一个名为 "main_app" 的可执行文件
add_executable(main_app src/main_app.cpp)

# 3. 链接 ROS 库和自定义库
target_link_libraries(main_app
  ${catkin_LIBRARIES}
  common_utils
)
```
配置方法：

1. 使用 `add_library()` 定义： 当你使用 `add_library(common_utils ...)` 创建库时，`common_utils` 就是这个库在 `CMakeLists.txt` 中的逻辑名称。
2. 直接引用： 在 `target_link_libraries` 中，你可以直接使用这个逻辑名称来链接你的自定义库。CMake 会自动处理库文件的路径和依赖关系。