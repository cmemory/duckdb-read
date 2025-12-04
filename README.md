# duckdb-read

下面把从你在命令行执行 duckdb -c "INSTALL postgres" 或在程序中通过 client API 调用 Execute/Query 开始，到把语句交给解析器、生成 LoadStatement、最后触发安装/加载逻辑的“从入口到解析”的完整路径列清楚，指出具体函数/文件以便你从这些入口开始跟踪代码。

CLI（duckdb -c "..."）的入口
可追踪入口是 tools/shell/shell.cpp 的程序主流程（wmain / main）。
命令行选项解析把 -c 对应到 shell 命令处理项（tools/shell/shell_command_line_option.cpp）。-c 的实现会调用 RunCommand<true>，进而调用 ShellState::RunInitialCommand(...)（命令解析/运行的入口）。
RunInitialCommand 最终会调用 ShellState::ExecuteSQL 或 ExecuteSQLSingleValue 去执行传入的 SQL 字符串。
关键文件/函数：
tools/shell/shell_command_line_option.cpp （定义 -c 选项并转到 RunCommand）
tools/shell/shell.cpp :: ShellState::RunInitialCommand / ShellState::ExecuteSQL / ShellState::ExecuteSQLSingleValue
流程概览（CLI） duckdb (wmain) -> 选项解析 (-c) -> RunCommand -> ShellState::RunInitialCommand -> ShellState::ExecuteSQL(...) -> 调用连接执行 SQL（见下一节）

客户端 API 的入口（嵌入式 / C++ / C）
C++ 嵌入式：你会创建 duckdb::DuckDB db(...); 然后 duckdb::Connection con(db); 再用 con.Query("INSTALL postgres") 或 con.Execute("INSTALL postgres")。
示例参考： examples/embedded-c++/main.cpp
关键类/方法（src 面向库的实现）在 duckdb::Connection（库内部的 Query/Execute 会把 SQL 交给解析/执行管线）。
C API：调用 duckdb_query(duckdb_connection connection, const char *query, duckdb_result *out)（src/main/capi/duckdb-c.cpp）。
关键文件： src/main/capi/duckdb-c.cpp :: duckdb_query
从 Connection 到解析器（共用路径）
不论是 CLI 还是 API，最终都把 SQL 字符串送到解析器管线：
在 shell 的 ExecuteSQL 中使用的是连接（duckdb::Connection）来提取语句：con.ExtractStatements(zSql)（shell 调用示例见 tools/shell/shell.cpp）。
Connection 内部会调用 Parser/Query 引擎去解析并生成语句对象（Statement）。
关键文件/函数：
tools/shell/shell.cpp :: ShellState::ExecuteSQL （展示如何从 shell 层把 SQL 交给连接）
C++/C API：duckdb::Connection::Query / duckdb_query（C API） -> 交到数据库执行引擎
解析阶段（把 "INSTALL postgres" 变成 LoadStatement / LoadInfo）
Parser 主入口： src/parser/parser.cpp :: ParseQuery / Parse
Parser 可能直接用内置解析器或在某些情形下（Postgres 语法）用 libpg_query（PostgresParser）。
Transformer 把解析树变为内部 Statement：
两条转换路径都存在（PEG transformer 用于 autocomplete/front-end，libpgquery 路径用于 runtime）：
extension/autocomplete/transformer/transform_load.cpp :: TransformInstallStatement（从 PEG parse tree -> LoadStatement/LoadInfo）
src/parser/transform/statement/transform_load.cpp :: 把 PGLoadStmt -> LoadStatement/LoadInfo（libpgquery 路径）
最终：生成 LoadStatement，包含 LoadInfo：
info->filename = "postgres"
info->load_type = LoadType::INSTALL（或 FORCE_INSTALL）
info->repository/version 等根据语句内容填充
绑定/生成计划（binder -> logical plan）
Binder 把 LoadStatement 转为逻辑算子（LogicalSimple(LOGICAL_LOAD)）：
文件： src/planner/binder/statement/bind_load.cpp
Binder 会设置输出类型和一些语句属性，然后交给执行器生成物理计划。
执行阶段（PhysicalLoad 的分支：INSTALL vs LOAD）
物理算子实现： src/execution/operator/helper/physical_load.cpp :: PhysicalLoad::GetDataInternal
如果 info->load_type 为 INSTALL / FORCE_INSTALL：
调用 ExtensionHelper::InstallExtension(context.client, info->filename, options)
若指定 repository 则走 InstallFromRepository 的路径
否则（LoadType::LOAD）：
调用 ExtensionHelper::LoadExternalExtension(context.client, info->filename)
也就是说，PhysicalLoad 是把解析结果映射到“安装”或“直接加载”具体实现的执行点。
安装与加载的具体实现（关键文件）
安装：
src/main/extension/extension_install.cpp 包含 InstallExtension、InstallFromRepository、DirectInstallExtension、InstallFromHttpUrl、写入本地扩展文件等逻辑。
加载：
src/main/extension/extension_load.cpp 包含 TryInitialLoad、InitialLoad、LoadExternalExtension、LoadExternalExtensionInternal、权限检查、元数据解析和动态库符号查找/初始化逻辑。
自动安装：
InitialLoad 在 TryInitialLoad 失败且配置允许 autoinstall_known_extensions 时，会调用 InstallExtension 然后重试加载（这就是 LOAD 时可能触发自动安装的逻辑）。
扩展初始化（扩展如何被注册）
扩展二进制需要暴露约定的初始化函数（如 DUCKDB_CPP_EXTENSION_ENTRY 宏生成的入口），加载逻辑会查找并调用该入口函数以注册扩展的表/函数/逻辑（举例： extension/tpcds/tpcds_extension.cpp）。
LoadExternalExtensionInternal 会通过动态加载（平台 API）查找符号并执行初始化，然后 info.FinishLoad(...) 完成注册。
快速定位（从入口开始依次查看的文件）

CLI 起点： tools/shell/shell.cpp（wmain / ShellState::ExecuteSQL）
命令行选项映射（-c）： tools/shell/shell_command_line_option.cpp
C API 起点： src/main/capi/duckdb-c.cpp :: duckdb_query
解析器入口： src/parser/parser.cpp
Transform -> LoadStatement： extension/autocomplete/transformer/transform_load.cpp 与 src/parser/transform/statement/transform_load.cpp
Binder： src/planner/binder/statement/bind_load.cpp
物理执行算子： src/execution/operator/helper/physical_load.cpp
安装实现： src/main/extension/extension_install.cpp
加载实现： src/main/extension/extension_load.cpp
扩展示例入口： extension//_extension.cpp （如 extension/tpcds/tpcds_extension.cpp）
简短回答（要点回顾）

如果你从命令行运行 duckdb -c "INSTALL postgres"：从 wmain -> 命令行选项处理 -> RunCommand -> ShellState::RunInitialCommand -> ShellState::ExecuteSQL -> 通过 Connection 把 SQL 送到解析器（Parser::ParseQuery）并按上面 pipeline 转为 LoadStatement -> Binder -> PhysicalLoad -> ExtensionHelper::InstallExtension。
如果你在程序里用 client API（C++/C）调用 con.Query("INSTALL postgres") 或 duckdb_query(...)：路径在逻辑上相同，只是入口是 duckdb::Connection::Query 或 C API 的 duckdb_query，而后续仍进入相同的解析/执行管线。
