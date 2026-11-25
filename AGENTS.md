# AGENTS.md — Wayfire

Wayfire 是基于 wlroots 0.20 的 3D Wayland 合成器（C++17，Meson/Ninja，MIT）。本文件面向在本仓库内改代码的 agent。

## 目录与职责

- `src/core/`：核心合成器——`core.cpp`（启动/全局状态）、`plugin-loader.cpp`、`output-layout.cpp`、`scene.cpp`、`opengl.cpp`、`seat/`（输入）、`txn/`（事务）。
- `src/view/`：窗口抽象，`xdg-shell/`、`xwayland/`、`layer-shell/` 三种实现；`toplevel-view.cpp`、`view.cpp` 为公共路径。
- `src/output/`：输出、渲染管理、工作区流、自适应重绘调度。
- `src/api/wayfire/`：**对外安装的插件 SDK**（头文件会被外部插件使用），场景图/事务/信号等接口都在这里。
- `plugins/`：内置插件分组；`single_plugins/` 放单文件插件，`common/` 为共享代码；`plugins/meson.build` 维护加载列表。
- `metadata/*.xml`：每个插件的选项元数据（插件名、类型、默认值），运行时按此校验选项；新增选项必须同时改这里并加入 `metadata/meson.build`。
- `test/`：doctest 单测与协议测试；`test/support/` 是无头测试基础设施。
- `proto/`（自定义 wayland 协议）、`shaders/`（Vulkan effects）、`wayfire.ini`（默认配置样例）、`man/wayfire.1.in`（CLI 手册）。

## 构建 / 测试 / 格式化

```sh
git submodule update --init --recursive     # wlroots、wf-config、wf-utils、wf-touch、wf-json
meson setup build                           # 系统缺 wlroots/wf-config 时自动用 subprojects
ninja -C build
meson setup build -Dtests=enabled           # 测试需要 doctest；CI 也显式打开
ninja -C build test                         # 等价 meson test -C build
```

- 系统 wlroots 必须是 0.20.x、wf-config 0.12.x；`-Duse_system_wlroots=disabled/enabled` 可强制走子项目或系统包。
- 相关选项：`-Dtests`、`-Dvulkan_effects`、`-Dcustom_pch`、`-Dxwayland`。
- 单测：`meson test -C build --print-errorlogs "Xdg-shell test"`；直接跑 doctest 可执行文件时用 `--test-case=模式`。
- 开发环境运行：`meson devenv -C build wayfire -d`（自动设置 `WAYFIRE_PLUGIN_PATH`、`WAYFIRE_PLUGIN_XML_PATH`）。手工运行时这两个变量需指向 build 的插件目录与 `metadata/`。
- 格式化（CI 会检查 diff 为零）：`git ls-files | grep "hpp$\|cpp$" | xargs uncrustify -c uncrustify.ini --no-backup`，uncrustify 需 >= 0.71。缩进 4 空格、LF；`metadata/*.xml` 用 tab。
- CI（`.github/workflows/ci.yaml`）：Alpine gcc/musl、Arch clang/libc++ 两套完整构建 + `ninja test` + uncrustify 风格检查。改完至少在本地跑 `ninja -C build test`。

## 架构边界（编辑时必须遵守）

- **公共 API/ABI**：`src/api/wayfire/**` 是已安装头文件。破坏性改动需同步递增 `plugin.hpp` 中的 `WAYFIRE_API_ABI_VERSION_MACRO`（Meson 会读取它写入 pkg-config/构建信息），否则外部插件版本不匹配。
- **插件入口**：实现 `wf::plugin_interface_t`（`init()`/`fini()`），用 `DECLARE_WAYFIRE_PLUGIN(Class)` 导出 `newInstance()` 与 `getWayfireVersion()`；在 `plugins/meson.build`（或 `single_plugins/meson.build`）的列表登记，并在 `metadata/` 加 XML。`fini()` 必须断开信号、注销 binding、释放全局状态。
- **渲染与输入走场景图**：`src/api/wayfire/scene.hpp` 描述完整渲染/输入树。自定义效果加 render node；输入 grab 用覆盖全输出的 input-only node，不要再使用 0.8 之前的 custom renderer / grab 接口。
- **Toplevel 状态变更走事务**：`src/api/wayfire/txn/*`（`transaction_t` + `transaction_object_t`）保证多属性原子应用；直接改状态容易与客户端 ack 时序冲突。
- **事件用信号**：`wf::signal::connection_t<T>` + `connect()`，信号定义在 `signal-definitions.hpp`。
- **插件只依赖公共头**：插件代码 include `wayfire/...`，不要 include `src/core/*-impl.hpp` 等内部实现；核心内部头只给 `src/` 内使用。
- **wlroots**：pinned 0.20，全局 `-DWLR_USE_UNSTABLE`。include 用 `<wayfire/nonstd/wlroots-full.hpp>`；wlroots 的 unstable API 变动需与子模块版本匹配。
- **输出作用域**：per-output 插件用 `wf::per_output_plugin_instance_t` / `per_output_tracker_mixin_t`，不要假设单输出。
- **日志**：使用 `<wayfire/util/log.hpp>` 的 `LOGE/LOGW/LOGI/LOGD`；高频/分类日志用 `LOGC(CATEGORY, ...)`（分类见 `debug.hpp` 的 `logging_category`），避免裸 `printf`/`std::cout`。
- **CLI 参数**：新增选项需同时改 `src/main.cpp` 的 `print_help()` 与 `man/wayfire.1.in`。

## 测试约定

- 测试用 doctest，在 `test/<area>/meson.build` 中建可执行文件并 `test('名称', exe)` 注册；可执行文件链接静态库 `libwayfire`。
- 协议测试（`test/protocol/`）通过 `test/support/headless-core-harness.*` 启动 **wlroots headless 后端**，并用真实 Wayland 客户端（`wayland-xdg-client.*`、`wayland-layer-shell-client.*`）交互；不需要 GPU/显示服务器，harness 自行设置 `XDG_RUNTIME_DIR`/`WAYLAND_DISPLAY`。
- 协议测试通过编译宏 `TEST_METADATA_DIR`、`TEST_DEFAULTS_INI` 读取源码树中的 `metadata/` 与 `wayfire.ini`；新增同类测试请沿用。
- **坑**：`wf::option_wrapper_t` 的静态实例按进程缓存选项值；需要同一选项不同值的测试必须拆成不同可执行文件（`test/protocol/meson.build` 中有先例）。

## 其他

- 外部插件管理文档：`docs/wayfire-plugin.md`、`docs/wayfire-plugin-author.md`，工具为 `scripts/wayfire-plugin.py`。
- 默认插件列表在 `metadata/core.xml` 的 `core/plugins`；修改默认插件集要同步 `wayfire.ini` 与相关 XML。
- 当前分支 `master`，上游为 `WayfireWM/wayfire`；提交保持一提交一件事。
- 版本：`meson.build` 以最近发布版为基线（当前 0.11.0）；git 快照由 `wayfire --version` 的提交号和打包 workflow 的 `0.11.0+git<日期>.<提交>` 标识。
