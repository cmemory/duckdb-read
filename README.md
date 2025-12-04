下面把从你在命令行执行 duckdb -c "INSTALL postgres" 或在程序中通过 client API 调用 Execute/Query 开始，到把语句交给解析器、生成 LoadStatement、最后触发安装/加载逻辑的“从入口到解析”的完整路径列清楚，指出具体函数/文件以便你从这些入口开始跟踪代码。
1. CLI（duckdb -c "..."）的入口
● 可追踪入口是 tools/shell/shell.cpp 的程序主流程（wmain / main）。
● 命令行选项解析把 -c 对应到 shell 命令处理项（tools/shell/shell_command_line_option.cpp）。-c 的实现会调用 RunCommand<true>，进而调用 ShellState::RunInitialCommand(...)（命令解析/运行的入口）。
● RunInitialCommand 最终会调用 ShellState::ExecuteSQL 或 ExecuteSQLSingleValue 去执行传入的 SQL 字符串。
    ○ 关键文件/函数：
        ■ tools/shell/shell_command_line_option.cpp （定义 -c 选项并转到 RunCommand）
        ■ tools/shell/shell.cpp :: ShellState::RunInitialCommand / ShellState::ExecuteSQL / ShellState::ExecuteSQLSingleValue
流程概览（CLI） duckdb (wmain) -> 选项解析 (-c) -> RunCommand -> ShellState::RunInitialCommand -> ShellState::ExecuteSQL(...) -> 调用连接执行 SQL（见下一节）
1. 客户端 API 的入口（嵌入式 / C++ / C）
● C++ 嵌入式：你会创建 duckdb::DuckDB db(...); 然后 duckdb::Connection con(db); 再用 con.Query("INSTALL postgres") 或 con.Execute("INSTALL postgres")。
    ○ 示例参考： examples/embedded-c++/main.cpp
    ○ 关键类/方法（src 面向库的实现）在 duckdb::Connection（库内部的 Query/Execute 会把 SQL 交给解析/执行管线）。
● C API：调用 duckdb_query(duckdb_connection connection, const char query, duckdb_result out)（src/main/capi/duckdb-c.cpp）。
    ○ 关键文件： src/main/capi/duckdb-c.cpp :: duckdb_query
1. 从 Connection 到解析器（共用路径）
● 不论是 CLI 还是 API，最终都把 SQL 字符串送到解析器管线：
    ○ 在 shell 的 ExecuteSQL 中使用的是连接（duckdb::Connection）来提取语句：con.ExtractStatements(zSql)（shell 调用示例见 tools/shell/shell.cpp）。
    ○ Connection 内部会调用 Parser/Query 引擎去解析并生成语句对象（Statement）。
    ○ 关键文件/函数：
        ■ tools/shell/shell.cpp :: ShellState::ExecuteSQL （展示如何从 shell 层把 SQL 交给连接）
        ■ C++/C API：duckdb::Connection::Query / duckdb_query（C API） -> 交到数据库执行引擎
1. 解析阶段（把 "INSTALL postgres" 变成 LoadStatement / LoadInfo）
● Parser 主入口： src/parser/parser.cpp :: ParseQuery / Parse
    ○ Parser 可能直接用内置解析器或在某些情形下（Postgres 语法）用 libpg_query（PostgresParser）。
● Transformer 把解析树变为内部 Statement：
    ○ 两条转换路径都存在（PEG transformer 用于 autocomplete/front-end，libpgquery 路径用于 runtime）：
        ■ extension/autocomplete/transformer/transform_load.cpp :: TransformInstallStatement（从 PEG parse tree -> LoadStatement/LoadInfo）
        ■ src/parser/transform/statement/transform_load.cpp :: 把 PGLoadStmt -> LoadStatement/LoadInfo（libpgquery 路径）
● 最终：生成 LoadStatement，包含 LoadInfo：
    ○ info->filename = "postgres"
    ○ info->load_type = LoadType::INSTALL（或 FORCE_INSTALL）
    ○ info->repository/version 等根据语句内容填充
1. 绑定/生成计划（binder -> logical plan）
● Binder 把 LoadStatement 转为逻辑算子（LogicalSimple(LOGICAL_LOAD)）：
    ○ 文件： src/planner/binder/statement/bind_load.cpp
● Binder 会设置输出类型和一些语句属性，然后交给执行器生成物理计划。
1. 执行阶段（PhysicalLoad 的分支：INSTALL vs LOAD）
● 物理算子实现： src/execution/operator/helper/physical_load.cpp :: PhysicalLoad::GetDataInternal
    ○ 如果 info->load_type 为 INSTALL / FORCE_INSTALL：
        ■ 调用 ExtensionHelper::InstallExtension(context.client, info->filename, options)
        ■ 若指定 repository 则走 InstallFromRepository 的路径
    ○ 否则（LoadType::LOAD）：
        ■ 调用 ExtensionHelper::LoadExternalExtension(context.client, info->filename)
● 也就是说，PhysicalLoad 是把解析结果映射到“安装”或“直接加载”具体实现的执行点。
1. 安装与加载的具体实现（关键文件）
● 安装：
    ○ src/main/extension/extension_install.cpp 包含 InstallExtension、InstallFromRepository、DirectInstallExtension、InstallFromHttpUrl、写入本地扩展文件等逻辑。
● 加载：
    ○ src/main/extension/extension_load.cpp 包含 TryInitialLoad、InitialLoad、LoadExternalExtension、LoadExternalExtensionInternal、权限检查、元数据解析和动态库符号查找/初始化逻辑。
● 自动安装：
    ○ InitialLoad 在 TryInitialLoad 失败且配置允许 autoinstall_known_extensions 时，会调用 InstallExtension 然后重试加载（这就是 LOAD 时可能触发自动安装的逻辑）。
1. 扩展初始化（扩展如何被注册）
● 扩展二进制需要暴露约定的初始化函数（如 DUCKDB_CPP_EXTENSION_ENTRY 宏生成的入口），加载逻辑会查找并调用该入口函数以注册扩展的表/函数/逻辑（举例： extension/tpcds/tpcds_extension.cpp）。
● LoadExternalExtensionInternal 会通过动态加载（平台 API）查找符号并执行初始化，然后 info.FinishLoad(...) 完成注册。
快速定位（从入口开始依次查看的文件）
● CLI 起点： tools/shell/shell.cpp（wmain / ShellState::ExecuteSQL）
● 命令行选项映射（-c）： tools/shell/shell_command_line_option.cpp
● C API 起点： src/main/capi/duckdb-c.cpp :: duckdb_query
● 解析器入口： src/parser/parser.cpp
● Transform -> LoadStatement： extension/autocomplete/transformer/transform_load.cpp 与 src/parser/transform/statement/transform_load.cpp
● Binder： src/planner/binder/statement/bind_load.cpp
● 物理执行算子： src/execution/operator/helper/physical_load.cpp
● 安装实现： src/main/extension/extension_install.cpp
● 加载实现： src/main/extension/extension_load.cpp
● 扩展示例入口： extension//_extension.cpp （如 extension/tpcds/tpcds_extension.cpp）
简短回答（要点回顾）
● 如果你从命令行运行 duckdb -c "INSTALL postgres"：从 wmain -> 命令行选项处理 -> RunCommand -> ShellState::RunInitialCommand -> ShellState::ExecuteSQL -> 通过 Connection 把 SQL 送到解析器（Parser::ParseQuery）并按上面 pipeline 转为 LoadStatement -> Binder -> PhysicalLoad -> ExtensionHelper::InstallExtension。
● 如果你在程序里用 client API（C++/C）调用 con.Query("INSTALL postgres") 或 duckdb_query(...)：路径在逻辑上相同，只是入口是 duckdb::Connection::Query 或 C API 的 duckdb_query，而后续仍进入相同的解析/执行管线。





总体位置
● CLI 源文件： tools/shell/shell.cpp
● 命令行选项到 handler 的映射： tools/shell/shell_command_line_option.cpp （你可以在这两个文件里从 main 开始向下追踪所有逻辑）
一、程序启动（main 的职责）
1. 解析命令行参数（argv）
    a. main 会解析命令行选项并把它们转换成内部的 command_line_calls / extra_commands 等数据结构。
    b. 命令行选项表定义在 tools/shell/shell_command_line_option.cpp，中间把短选项如 -c 映射到对应的处理器（在该文件中 -c 对应 RunCommand<true>）。
2. 初始化 ShellState（程序全局/会话状态）
    a. 创建 ShellState（包含配置 DBConfig、是否安全模式 safe_mode、初始化文件 initFile、数据库文件名等）。
    b. 可能会读取 init 文件（ProcessDuckDBRC），再做第二次命令行回调（有些选项在 init 之后才应用）。
3. 对 -c 选项的处理（关键点）
    a. 在命令行选项解析中，如果检测到 -c "COMMAND"（在 options 表中 "c"），它会调用 RunCommand<true>，该模板会调用： RunCommand -> ShellState::RunInitialCommand(cmd.c_str(), bail) 其中 bail = true for -c（表示执行失败则退出）。
    b. RunInitialCommand 是命令运行的入口：它会把命令区分为 meta-command（以 '.' 开头）或 SQL 命令，并调用对应处理函数（例如 ExecuteSQL）。
二、RunInitialCommand / ExecuteSQL（把 SQL 送入数据库）
● ShellState::RunInitialCommand(...)（tools/shell/shell.cpp）
    ○ 如果命令以 '.' 开头走 DoMetaCommand，否则把 SQL 字符串交给 ExecuteSQL。
● ShellState::ExecuteSQL(const string &query)（tools/shell/shell.cpp）
    ○ 通过当前的 Connection（conn），把 SQL 交给 con.Query 或通过 ExtractStatements 拆成多个 statement 并逐个执行（代码片段中 con.ExtractStatements(zSql)）。
    ○ 在执行之前会根据 echo/模式等做一些打印设置（如果 -echo 打开会先打印 SQL）。
断点建议（从 main 起逐步跟踪）
● 在 tools/shell/shell_command_line_option.cpp 中找到 -c 的选项定义处，设置断点，确认命令字符串被正确传入。
● 在 ShellState::RunInitialCommand 上设置断点（确认它把命令识别为 SQL 并调用 ExecuteSQL）。
● 在 ShellState::ExecuteSQL 或其内部的 ExecuteQuery / ExecuteSQLSingleValue 上设置断点（看到它如何把 SQL 交给 Connection）。
三、Connection / Query 执行（进入解析器）
● con.Query / Connection::Query（C++ API）或 C API duckdb_query（src/main/capi/duckdb-c.cpp）是程序执行 SQL 的入口。
    ○ C API 的 duckdb_query 最终会调用绑定到 Connection 的 Query 方法。
● Connection::Query 会触发查询管线：解析 -> 语法树转换 -> 绑定 -> 优化/规划 -> 执行。
    ○ 解析器主入口函数在 src/parser/parser.cpp（ParseQuery），可以在这里下断点来查看解析树。
    ○ 如果 SQL 包含扩展语句（INSTALL/LOAD），parser 会生成 LoadStatement / LoadInfo（transform 位于 extension/autocomplete/transformer/transform_load.cpp 和 src/parser/transform/statement/transform_load.cpp）。
断点建议（进入执行引擎）
● 在 Connection::Query 的实现处断点（IDE 中搜索 Connection::Query）。
● 在 src/parser/parser.cpp 的 ParseQuery 处断点（观察 ParseResult）。
● 在 transform_load.cpp 或 src/parser/transform/statement/transform_load.cpp 下断点（观察如何把解析树转成 LoadStatement/LoadInfo）。
四、Bind -> 规划 -> 执行（执行 INSTALL 流程）
● 绑定（Binder）阶段：LoadStatement 会被 Binder 转成逻辑算子（logical operator LOGICAL_LOAD）。
    ○ 绑定代码文件： src/planner/binder/statement/bind_load.cpp
● 物理执行阶段：对应的物理算子为 PhysicalLoad（src/execution/operator/helper/physical_load.cpp）。
    ○ PhysicalLoad::GetDataInternal 会根据 LoadInfo.load_type 分支：
        ■ INSTALL / FORCE_INSTALL -> 调用 ExtensionHelper::InstallExtension(...)
        ■ LOAD -> 调用 ExtensionHelper::LoadExternalExtension(...)
    ○ 因此，最终安装动作由 ExtensionHelper 的相关函数实现。
断点建议（追到安装实现）
● 在 src/execution/operator/helper/physical_load.cpp::PhysicalLoad::GetDataInternal 下断点（这里调用 InstallExtension）。
● 在 src/main/extension/extension_install.cpp 的 InstallExtension / InstallExtensionInternal / InstallFromRepository / DirectInstallExtension 下断点（跟踪下载/写入本地扩展目录的实现）。
● 在 src/main/extension/extension_load.cpp 的 TryInitialLoad / InitialLoad / LoadExternalExtension 下断点（跟踪载入、权限、自动安装重试逻辑）。
五、OpenDB / Shell 在打开数据库时的初始化（与 main 相关）
● 当 shell 需要打开数据库，它会调用 ShellState::OpenDB（tools/shell/shell.cpp）：
    ○ OpenDB 会创建 DuckDB 实例： db = make_uniqduckdb::DuckDB(zDbFilename.c_str(), &config);
    ○ 然后创建连接： conn = make_uniqduckdb::Connection(*db);
    ○ OpenDB 中还会加载一些静态扩展： db->LoadStaticExtension<AutocompleteExtension>(); db->LoadStaticExtension<ShellExtension>();
    ○ 如果 safe_mode 打开，会执行 SQL "SET enable_external_access=false"，限制扩展加载等。
断点建议
● 在 OpenDB（tools/shell/shell.cpp）设置断点，观察 DBConfig、safe_mode、静态扩展加载等。
六、示例：完整调用链（duckdb -c "INSTALL postgres"） 下面是建议的逐步追踪点（call stack 顺序）：
1. main（tools/shell/shell.cpp）启动并解析 argv
2. 命令行选项解析处（tools/shell/shell_command_line_option.cpp）发现 -c，调用 RunCommand<true>
3. RunCommand -> ShellState::RunInitialCommand(cmd, bail=true)
4. ShellState::RunInitialCommand -> ShellState::ExecuteSQL("INSTALL postgres")
5. ShellState::ExecuteSQL -> conn->Query("INSTALL postgres")（或先 ExtractStatements 再单语句执行）
6. Connection::Query -> Parser::ParseQuery (src/parser/parser.cpp)
7. Transformer（transform_load.cpp 或 transform_load via libpgquery）将 parse tree 转为 LoadStatement / LoadInfo
8. Binder::Bind(LoadStatement) -> 生成 LogicalSimple(LOGICAL_LOAD)
9. Planner/Executor -> 物理算子 PhysicalLoad 执行（src/execution/operator/helper/physical_load.cpp）
10. PhysicalLoad::GetDataInternal 检测是 INSTALL -> 调用 ExtensionHelper::InstallExtension(context.client, "postgres", options)
11. InstallExtension -> InstallExtensionInternal / InstallFromRepository / DirectInstallExtension 等（src/main/extension/extension_install.cpp）
12. 写入本地扩展文件、记录 metadata、完成安装。安装成功后（如果是 initial load 路径）可能会调用加载逻辑继续初始化扩展。
七、如果你想从 main 实际单步调试
● 在 tools/shell/shell.cpp 的 main（或程序入口）打断点，运行带参数： ./duckdb -c "INSTALL postgres"
● 逐步运行到 RunCommand、RunInitialCommand、ExecuteSQL、Connection::Query、ParseQuery、Transform、Binder、PhysicalLoad、InstallExtension 等关键点。
● 推荐的断点位置（文件 + 函数）：
    ○ tools/shell/shell_command_line_option.cpp :: RunCommand / mapping for "-c"
    ○ tools/shell/shell.cpp :: ShellState::RunInitialCommand, ShellState::ExecuteSQL, ShellState::OpenDB
    ○ src/main/capi/duckdb-c.cpp :: duckdb_query （如果你用 C API）
    ○ src/parser/parser.cpp :: ParseQuery
    ○ extension/autocomplete/transformer/transform_load.cpp :: TransformInstallStatement
    ○ src/parser/transform/statement/transform_load.cpp :: PG->LoadStatement 转换
    ○ src/planner/binder/statement/bind_load.cpp :: Binder::Bind(LoadStatement)
    ○ src/execution/operator/helper/physical_load.cpp :: PhysicalLoad::GetDataInternal
    ○ src/main/extension/extension_install.cpp :: InstallExtension / InstallExtensionInternal
    ○ src/main/extension/extension_load.cpp :: InitialLoad / TryInitialLoad / LoadExternalExtensionInternal
