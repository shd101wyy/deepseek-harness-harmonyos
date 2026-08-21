# DeepSeek Harness 在 HarmonyOS (MateBook Pro) 上的安装与排障记录

> 目标:在华为 MateBook Pro(HarmonyOS,内核 `HarmonyOS HongMeng Kernel 1.12.0`,AArch64)
> 上安装并运行 DeepSeek Harness (`@deepseek-ai/dsh`,Web UI 版)。
>
> 环境事实:
> - 包管理使用 **harmonybrew**(Homebrew 的 HarmonyOS 移植),前缀 `~/.harmonybrew`
> - 系统**没有 make、没有 Python**,需要自行安装
> - 文件系统是 **hmdfs**(华为分布式文件系统),**不支持硬链接**
> - npm 官方源 `registry.npmjs.org` 不可用(连接失败/curl 崩溃),必须用镜像

---

## 0. 一键脚本 dsh-hmos

本目录下的 `dsh-hmos` 脚本封装了全部安装、patch 与服务管理,已链接到
`~/.harmonybrew/bin/dsh-hmos`,可直接使用:

```bash
dsh-hmos install      # 检查 brew -> 安装依赖 -> 比对/安装最新版 dsh -> 打 HarmonyOS patch -> 构建原生模块(版本门控幂等;有新版自动升级)
dsh-hmos uninstall    # 卸载 dsh 及脚本状态(保留 ~/.dsh 用户数据)
dsh-hmos start [flags]  # 后台启动 dsh web;flags 原样透传给 dsh web
dsh-hmos status       # 查看所有后台实例运行状态
dsh-hmos stop [--port N]  # 停止指定端口实例;不带参数停止全部
dsh-hmos restart [--port N]  # 重启全部(或指定端口)实例,复用原启动参数
dsh-hmos exec ...     # 透传运行 dsh 命令,如: dsh-hmos exec --help
dsh-hmos help         # 帮助
```

`start` 支持 dsh web 的全部 flag,并**原样透传**:

```bash
dsh-hmos start                              # 默认 http://127.0.0.1:3080
dsh-hmos start --port 8080                  # 自定义端口
dsh-hmos start --host 0.0.0.0 --port 8080   # 绑定所有网卡
dsh-hmos start --trusted-host myhost:8080   # 额外信任的来源
```

**多实例**:不同 `--port` 可同时运行多个 web 实例,各自独立的
PID/日志文件(`~/.dsh-hmos/web-<port>.{pid,log}`);`status` 列出全部,
`stop --port N` 只停指定实例。注意:实例共享 `~/.dsh`(会话/凭据),
同时跑 agent 任务可能互相干扰。

**重启/关机场景**:start 后未 stop 直接关机,`~/.dsh-hmos/web-*.pid`
会残留。脚本以**端口探测为准**(PID 仅作辅助,且会校验
`/proc/<pid>/cmdline` 确属 dsh web),因此:
- `status` 能正确区分"运行中 / 启动中 / 未运行",残留 PID 文件自动清理
- `start` 不会误报"已在运行",会正常拉起服务
- `stop` 不会误杀 PID 被系统复用的无关进程

`restart` 重启正在运行的实例(不带参数 = 全部;`--port N` = 指定端口)。
它会复用启动时记录在 `~/.dsh-hmos/web-<port>.args` 的原始参数
(`--host`/`--trusted-host` 等),先停后启,并等待旧进程释放端口(最多
10 秒)再拉起,避免新实例端口绑定失败;早期版本启动的实例(无 `.args`
记录)退化为以默认参数重启该端口。

`install` 在 brew 未安装时会提示访问 https://harmonybrew.atomgit.com 并退出;
已安装则自动完成依赖(node ohos-sdk cmake make ninja python@3.14 ripgrep)、npm 安装
`@deepseek-ai/dsh`、四处 HarmonyOS 必需 patch(见下文第 4.2、7、8、10 节)以及
node-pty / koffi / sharp 原生模块构建,最后做原生模块加载自检(失败时自动清理并重建原生模块再验一次)。重复执行安全(幂等)。
`install` 采用**版本门控**:先以轻量元数据查询(`npm view @latest version`,约 2s)与
已装 `package.json` 版本比对——**未安装或有新版才执行 npm 安装**;已在最新版或
离线查询失败时**跳过 npm**,保留现有安装。这一点很关键:npm 的 reify 每次都会
重解包整棵树(同版本也会 `changed N packages`),若每次都重装,会覆盖 4 处 patch、
清掉 koffi/node-pty 构建产物、剪掉 extraneous 的 sharp-wasm32,等于把"幂等快速
重跑"变成"每次全量重装 5 分钟"。检测到新版安装后,node_modules 被整体替换,
4 处 patch 与原生产物照旧由脚本自动重打/重建——**升级 dsh 只需重跑
`dsh-hmos install`**,不再需要单独的 `reinstall` 命令(该子命令已移除)。
`uninstall` 会先停服务、卸载 npm 包并清理脚本状态目录(`~/.dsh-hmos`),
**保留** `~/.dsh`(会话/凭据/配置);彻底清除需手动删除 `~/.dsh`。

---

## 1. 依赖清单(harmonybrew 需要安装的包)

| 包 | 安装命令 | 用途 | 备注 |
|---|---|---|---|
| `node` | `brew install node` | dsh 运行时 | 本文写作时 v26.7.0 |
| `ohos-sdk` | `brew install ohos-sdk` | 提供 `clang`/`clang++` 编译器 | 版本 15.0.4 |
| `cmake` | `brew install cmake` | koffi 原生模块源码构建 | |
| `make` | `brew install make` | node-pty 构建(node-gyp 使用 make) | 本机实测必要,勿漏 |
| `ninja` | `brew install ninja` | koffi 的 cmake 生成器 | |
| `python@3.14` | `brew install python@3.14` | node-pty 构建(node-gyp 需要 Python) | 装完后 `python3` 在 `~/.harmonybrew/bin/python3` |
| `ripgrep` | `brew install ripgrep` | dsh 文件搜索工具(grep/glob) | 见第 10 节:npm 自带的 `@vscode/ripgrep` 在 openharmony 上解析不到平台包,脚本会把它回退到本包 |

> 备注:
> - `clang`/`clang++` 由 `ohos-sdk` 提供,路径 `~/.harmonybrew/bin/clang{,++}`。
> - `make` 与 `ninja` 都必需,用途不同:`make` 供 node-gyp 构建 node-pty,
>   `ninja` 供 cmake 构建 koffi(本机实测 harmonybrew 的 make 直接在
>   `~/.harmonybrew/bin/make`,不是 keg-only 的 gmake)。
> - GNU grep **非必需**(脚本与 dsh 只用 Toybox grep 也支持的基础参数
>   `-q`/`-E`/`-v`);如需可 `brew install grep`。注意 brew 没有 `-y` 参数
>   (那是 apt/yum 的写法),直接 `brew install <包名>` 即可。

---

## 2. 安装命令(最终成功路径)

### 2.1 安装 dsh(使用 npmmirror 镜像)

```bash
# 官方源不可用,必须 --registry 指定镜像
npm install -g @deepseek-ai/dsh --registry=https://registry.npmmirror.com
```

**第一次会失败**,原因见下文第 3、4 节(node-pty / koffi 原生模块)。成功路径是:

```bash
# 1) 先跳过所有构建脚本,把 524 个 JS 包装好
npm install -g @deepseek-ai/dsh --ignore-scripts --registry=https://registry.npmmirror.com

# 2) 手动构建 node-pty(见第 3 节;本机无 cc/gcc,必须显式指定 clang)
cd ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/node_modules/node-pty
export CC=~/.harmonybrew/bin/clang CXX=~/.harmonybrew/bin/clang++
node ~/.harmonybrew/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js rebuild

# 3) 手动构建 koffi(见第 4 节,必须先用 toolchain + 禁用 strip)
cd ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/node_modules/koffi
export CC=~/.harmonybrew/bin/clang CXX=~/.harmonybrew/bin/clang++
node ./cnoke.cjs -P . -D src/koffi --release \
  -dCMAKE_TOOLCHAIN_FILE=/绝对路径/ohos-aarch64-toolchain.cmake

# 4) sharp 用 WebAssembly 版(见第 5 节)
cd ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh
npm install @img/sharp-wasm32 --ignore-scripts --registry=https://registry.npmmirror.com --no-save

# 5) 文件搜索工具回退到系统 ripgrep(见第 10 节)
brew install ripgrep
#    patch ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/node_modules/@vscode/ripgrep/lib/index.js
#    (或直接跑 dsh-hmos install 自动完成)
```

### 2.2 启动 Web UI

```bash
# NODE_OPTIONS 不允许 --expose-internals,必须直接传给 node
node --expose-internals ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/lib/bin.js web
# 启动后访问 http://127.0.0.1:3080
```

已封装为脚本 `~/.harmonybrew/bin/dsh-web`(内容就是上面这条命令)。

---

## 3. 问题:node-pty 构建失败 "Could not find any Python installation"

**症状**

```text
gyp ERR! find Python Could not find any Python installation to use
gyp ERR! System HarmonyOS HongMeng Kernel 1.12.0
```

**根因**:node-pty(pty 原生模块)没有 HarmonyOS 预编译二进制,node-gyp 必须
从源码构建,而 node-gyp 需要 Python;系统没有 Python。

**解决**

```bash
brew install python@3.14
# 另:本机没有 cc/gcc(只有 ohos-sdk 的 clang),构建前显式指定:
# export CC=~/.harmonybrew/bin/clang CXX=~/.harmonybrew/bin/clang++
# (make 内置默认 CC=cc,不显式指定时 C 编译会报 cc: command not found)
```

---

## 4. 问题:koffi 构建(最复杂,两个子问题)

### 4.1 cmake 不认识 HarmonyOS → 错误地编译 x86-64 汇编

**症状**

```text
-- SYSTEM_NAME=HarmonyOS
-- SYSTEM_PROCESSOR=unknown
FAILED: CMakeFiles/koffi.dir/src/abi/x64sysv_asm.S.obj
error: unknown token in expression   movq %rdi, %r11
```

**根因**:koffi 的 `CMakeLists.txt` 用 `CMAKE_SYSTEM_PROCESSOR MATCHES "aarch|arm"`
选择 AArch64 汇编文件;但 cmake 不认识 HarmonyOS,`CMAKE_SYSTEM_PROCESSOR=unknown`,
于是 fallback 到 x86-64 分支,在 arm64 上用 clang 汇编 x86 代码 → 报错。
(`-DCMAKE_SYSTEM_PROCESSOR=aarch64` 命令行传参不生效,它是 cmake 内部只读变量。)

**解决**:写一个 toolchain 文件强制 cmake 认为自己是 Linux/aarch64,通过
cnoke 的 `-d` 参数传入:

```cmake
# ohos-aarch64-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
execute_process(COMMAND uname -m OUTPUT_VARIABLE HOST_ARCH OUTPUT_STRIP_TRAILING_WHITESPACE)
set(CMAKE_SYSTEM_PROCESSOR ${HOST_ARCH})
```

```bash
node ./cnoke.cjs -P . -D src/koffi --release \
  -dCMAKE_TOOLCHAIN_FILE=/绝对路径/ohos-aarch64-toolchain.cmake
```

### 4.2 dlopen 拒绝加载被 strip 过的 .node 文件

**症状**(构建成功后加载仍失败)

```text
Error: Error loading shared library .../koffi.node: Permission denied
code: 'ERR_DLOPEN_FAILED'
```

**排查过程**(对照实验)

- `node-pty/build/Release/pty.node`(未 strip)能加载;koffi.node 不能;
- 把 koffi.node 复制到 pty 的目录仍失败 → 与路径无关,是文件本身问题;
- ELF 头、依赖(`libc++_shared.so`、`libc.so`)、section 布局都一致;
- 最小复现:编译一个 hello-world `.node`,各种编译选项(-O3、-fno-exceptions、
  -fno-rtti、-fvisibility=hidden、--gc-sections、C++20)都能加载;
- **唯一区别**:koffi 构建后执行了 `strip`(CMakeLists.txt 的 POST_BUILD),
  没有 `.symtab`;`llvm-strip` 处理 hello.node 后同样加载失败 → 结论:
  **HarmonyOS 的 dlopen 拒绝加载 stripped 的共享库**。

**解决**:修改 koffi 的 `src/koffi/CMakeLists.txt`,注释掉 POST_BUILD strip 块:

```cmake
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    # strip DISABLED for HarmonyOS: stripped .node fails to dlopen
    # (Permission denied). Keep the symbol table so the loader accepts it.
endif()
```

然后 `rm -rf build` 重新完整构建。

> ⚠️ 该修改在 `node_modules` 里,重新 `npm install`/升级 dsh 后会被覆盖,需要重打补丁。
> 官方预构建包 `@koromix/koffi-linux-arm64`(含 musl 版)在 HarmonyOS 上同样
> 无法加载(也是 strip 过的),所以必须本地源码构建。

---

## 5. 问题:sharp 模块 "Could not load the sharp module using the openharmony-arm64 runtime"

**根因**:sharp 的原生二进制不支持 openharmony-arm64。

**解决**(官方错误提示给出的方案):

```bash
npm install @img/sharp-wasm32 --registry=https://registry.npmmirror.com --no-save
```

验证:生成 4x4 PNG 成功即 OK。

---

## 6. 问题:启动时报 "--expose-internals is required for HMR service"

**根因**:`@deepseek-ai/cordis-plugin-hmr`(热重载服务)需要 node 的
`--expose-internals` 标志。

**坑**:`NODE_OPTIONS="--expose-internals"` 会被 node 拒绝
(`--expose-internals is not allowed in NODE_OPTIONS`)。

**解决**:直接把它作为 node 参数传入:

```bash
node --expose-internals ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/lib/bin.js web
```

---

## 7. 问题:会话保存失败 EPERM link

**症状**

```text
EPERM: operation not permitted, link
'.../.dsh/sessions/<project>/session-<id>/session.jsonl.zstd.<tmp>' ->
'.../.dsh/sessions/<project>/session-<id>/session.jsonl.zstd'
```

**根因**:`@deepseek-ai/dsh-session-persistence-jsonl` 的 `materializePosix()`
用 **硬链接** 做原子发布(`await link(tmp, finalPath)` 后删除 tmp)。
HarmonyOS 的 hmdfs 文件系统**不支持 link(2)**,返回 EPERM。

**定位**:
`~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-session-persistence-jsonl/lib/index.js`
第 1128 行(全文件唯一一处 `link(`)。

**解决**:把 `link(tmp, finalPath)` 替换为同文件系统内同样原子的
`rename(tmp, finalPath)`(Windows 路径本来就用的 rename 系实现,不回归)。

```js
// import 行:link → rename
import { rename, mkdir, mkdtemp, open, readFile, readdir, realpath, rm, stat, truncate } from "node:fs/promises";
// 第 1128 行
await rename(tmp, finalPath);
```

> ⚠️ 同样是 node_modules 内的补丁,升级 dsh 后需重打。

---

## 8. 问题:凭据文件权限检查失败(mode 660,chmod 无效)

**症状**(修复 link 后重启服务时)

```text
credentials-local: /storage/Users/currentUser/.dsh/.credentials.yaml is readable
beyond its owner (mode 660); run "chmod 600 ..." before starting again
```

**根因**:`dsh-credentials-local` 启动时校验凭据文件权限必须是 `0600`
(`mode & 0o077 == 0`)。但 **hmdfs 上 chmod 无效**——所有文件权限固定为
`660`(owner 20001006:file_manager),`chmod 600` 后 `stat` 仍是 660。
这是华为分布式文件系统的权限模型,用户无法改变。

**解决**:patch `dsh-credentials-local`,在 openharmony 平台跳过该检查
(该平台权限由系统管理,chmod 无意义):

```js
// ~/.harmonybrew/.../dsh-credentials-local/lib/index.js 第 88 行
if (process.platform === "win32" || process.platform === "openharmony") return; // hmdfs: mode is always 660 and chmod is a no-op
```

> 验证:`chmod 600` 后 `stat -c "%a"` 仍返回 660,证明 hmdfs 忽略 chmod。

---

## 9. 问题:沙箱后端不可用(SANDBOX_UNAVAILABLE)

**症状**(在 web UI 中发起对话/执行 bash 工具时)

```text
Error: sandbox mode "workspace-write" is requested but no sandbox backend is
usable on this host; refusing to run the command unconfined. Install bubblewrap
or run a Landlock-enforcing kernel (Linux), ...
```

**根因**:dsh 的进程沙箱(`dsh-sandbox-local`)在 Linux 上的后端链是
`bwrap` → `landlock`,按平台探测选择。**HarmonyOS 上两者都不可用**:

- `bwrap`:依赖 user namespaces,但 HarmonyOS 内核禁止:
  `unshare --user` 直接 `Bad system call`;bwrap 报
  `Unexpected capabilities but not setuid, old file caps config?`(无 CAP_SYS_ADMIN,
  CapEff 只有 0x2a)
- `landlock`:内核未启用 Landlock LSM(`/sys/kernel/security/lsm` 为空)

于是任何受限模式(`read-only` / `workspace-write`)的 bash 调用都 fail-closed,
抛出 `SANDBOX_UNAVAILABLE`。

**解决**:利用官方提供的部署开关——`dsh-base` 的 bundle 配置:

```yaml
- id: sandbox-policy
  config:
    mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'
```

`danger-full-access` 模式下消费方**直接 spawn 原始 argv,不调用 `ctx.sandbox`**
(见 `docs/subsystems/sandbox.zh.md`),所以只需在启动时设置:

```bash
export DSH_PERMISSION_MODE="danger-full-access"
node --expose-internals ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/lib/bin.js web
```

副作用(可接受):`dsh-base` 还会把权限审批从 `ask` 切到 `never`
(第 191 行 `policy: !!js ... === 'danger-full-access' ? 'never' : 'ask'`),
即工具调用不再逐次询问用户。在这台机器上系统本身权限模型受限
(hmdfs、无 user namespaces),风险可控。

> 已写入启动脚本 `~/.harmonybrew/bin/dsh-web`。
> ⚠️ 源码位置:排查依据
> `deepseek-harness/packages/bundle/base/cordis.patch.yml` 与
> `deepseek-harness/packages/sandbox/sandbox-policy/src/index.ts`。

---

## 10. 问题:文件搜索工具找不到 ripgrep(SEARCH_FAILED)

**症状**(agent 调用 grep/glob 搜索工具时)

```text
Could not find @vscode/ripgrep-openharmony-arm64. Ensure optionalDependencies
are installed for this platform (openharmony-arm64).
```

**根因**:dsh 的文件搜索工具(`dsh-tool-fs-search`)不调用系统 `rg`,而是
通过 `@vscode/ripgrep` 元包按 `process.platform` 拼出平台包
`@vscode/ripgrep-<platform>-<arch>` 并加载其自带二进制。openharmony 平台
**没有对应的平台包**;官方 `@vscode/ripgrep-linux-arm64` 的静态二进制又是
**strip 过的 ELF,hmdfs 拒绝 exec**(与第 4.2 节 koffi 被 strip 后 dlopen 被
拒是同一规律:本机实测 chmod +x 后执行仍 `Permission denied`)。

**解决**:brew 安装原生 ripgrep(已在依赖表),并 patch
`@vscode/ripgrep/lib/index.js`,让 openharmony 平台回退到 harmonybrew 的
`rg`(候选同时包含 `~/.harmonybrew/bin/rg` 与安装时解析出的绝对路径,
后者保证 HOME 被改写时仍能找到):

```js
} catch {
    const candidates = [
        `${require('node:os').homedir()}/.harmonybrew/bin/rg`,
        '/storage/Users/currentUser/.harmonybrew/bin/rg', // 安装时烘焙的绝对路径
    ];
    if (process.platform === 'openharmony') {
        resolved = candidates.find((p) => require('node:fs').existsSync(p));
    }
    if (!resolved) {
        throw new Error(/* 原错误 */);
    }
}
```

该解析是**惰性**的(首次搜索调用时才发生),所以 patch 后无需重启正在运行的
实例即可生效;`dsh-hmos install` 已自动打此补丁(幂等)。

> ⚠️ 同样是 node_modules 内的补丁,升级 dsh 后需重打(重跑 `dsh-hmos install` 即可)。
> 注意:系统 ripgrep 装好但**不打此补丁时搜索工具仍然不可用**——dsh 不会自动
> 发现系统 `rg`。

---

## 11. 附:验证命令速查

```bash
# 环境就绪检查
command -v node npm npx cmake make ninja python3 clang clang++ rg

# koffi 是否可加载(应输出 LOADED OK)
node -e "require('~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh/node_modules/koffi/build/koffi/openharmony_arm64/koffi.node')"

# sharp 是否可加载
node -e "require('sharp')"

# Web UI 是否存活
node -e "fetch('http://127.0.0.1:3080').then(r=>console.log(r.status))"

# ripgrep 解析是否已 patch(应输出 ~/.harmonybrew/bin/rg)
cd ~/.harmonybrew/lib/node_modules/@deepseek-ai/dsh
node --input-type=module -e "import('@vscode/ripgrep').then(m=>console.log(m.rgPath))"
```

---

## 12. 参考:另一位用户的部署方案(未采纳,备查)

另一位鸿蒙 PC 用户在 AtomGit 上发布了部署记录,与本方案对比后**决定不采纳**(截至
2026-08-14 本机方案已验证可用;如需简化 install 可随时切换)。

- 仓库:https://gitcode.com/u010189254/dsh-harmonyos-deploy
- 设备:鸿蒙 PC 7.0 / API 26 / HongMeng Kernel 1.13.0(本机 1.12.0)
- 与我们的关键差异(同为 `dsh 0.1.0-rc.6`):

| 项目 | 对方方案 | 本方案 | 结论 |
|---|---|---|---|
| koffi | **不构建**。patch `dsh-sandbox-local` 第 10 行,把对 `dsh-sandbox-windows-acl` 的顶层 import 改为 win32 条件惰性 `await import()`——koffi 只在 win32 后端被引用,非 win32 永不加载 | 源码构建(本 README 第 4.2 节) | 对方方案 install 更轻(免 cmake/ninja/toolchain/构建),升级后只需重放小补丁;但需改 node_modules,且 sandbox 插件将彻底无 koffi。**本机已构建成功且验证可用,保持现状**;若将来简化,补丁已确认与对方脚本 `reapply-koffi-patch.sh` 完全匹配 |
| 启动 | `setsid` 脱离进程组 + HTTP 探活 + mkdir 原子锁(弃用 flock:鸿蒙系统 PATH 无 flock) | `nohup` + PID 文件 + 端口探测(`dsh-hmos`) | 对方的 setsid + mkdir 锁更抗环境差异,可借鉴 |
| 开机自启 | 4 层钩子(/etc/profile 系统级、.zshenv、.zshrc、XDG autostart)+ 幂等探活;实测 HiShell 自启有 1~3 分钟延迟 | 无(手动 `dsh-hmos start`) | 暂不需要 |

- 对方踩坑记录中值得注意的环境事实(本机已确认不适用或已规避):
  - 鸿蒙 Toybox 限制:`head -n -5` 不支持、`/tmp` 只读、`whoami`/`id` 缺失、`sudo -n`/`-i` 报错 → 脚本只用系统路径工具
  - `df` 会把 `/storage/Users` 显示为 tmpfs 12GB 覆盖挂载,实际 hmfs 持久分区 ~450GB(重启不丢数据)
  - `~/.hdc/` 调试日志每小时 ~100MB,需定期清理
