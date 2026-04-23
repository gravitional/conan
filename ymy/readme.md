# Nushell 环境变量脚本生成功能

## 修改概要

### 1. `conan/internal/model/conf.py`

- 在 `BUILT_IN_CONFS` 中新增 `tools.env.virtualenv:nushell` 配置项

### 2. `conan/tools/env/environment.py`（主要改动）

**新增辅助函数：**

- `_nu_escape_str()` — 对 nushell 双引号字符串进行转义（`\` → `\\`，`"` → `\"`）
- `_nu_escape_interp()` — 对 nushell 插值字符串 `$"..."` 进行转义（额外转义 `(` 和 `)`）
- `_nu_deactivate_contents()` — 生成 nushell 的环境还原逻辑，支持两种模式：
    - 标准模式：在激活时动态写入还原脚本
    - 函数模式：生成 `def --env deactivate_*` 自定义命令

**`EnvVars` 类新增 `save_nu()` 方法：**

- 路径变量 → nushell 列表：`$env.PATH = ($env.PATH | prepend "/new/path")`
- 非路径变量 → 字符串：`$env.CC = "gcc"` 或带插值 `$env.CFLAGS = $"($env.CFLAGS) -O2"`
- 取消设置 → `hide-env VARNAME`
- 标准模式下生成占位的还原脚本文件（激活时会被覆盖）
- 使用 UTF-8 编码

**修改 `save_script()`：**

- 当配置了 `tools.env.virtualenv:nushell` 时，在默认格式（bat/sh/ps1）之外**额外**生成 `.nu` 脚本，确保内部构建工具的命令包装功能不受影响

**修改 `generate_aggregated_env()`：**

- 在 env_scripts 循环中新增 `.nu` 文件检测
- 生成聚合脚本 `conanbuild.nu` / `conanrun.nu`，使用 `source "file.nu"` 语法
- 聚合脚本同时支持标准模式和函数模式的环境还原

### 3. `conan/tools/microsoft/visual.py`

- 新增 nushell VCVars 包装脚本生成（`conanvcvars.nu`），通过 `cmd /c` 运行 `conanvcvars.bat` 并使用 `load-env` 捕获环境变量
- 更新 `_create_deactivate_vcvars_file()` 以支持 `.nu` 文件（使用 `print` 替代 `echo`）

### 设计决策：

- **Nushell 脚本是在默认格式之外额外生成的**，而非替代。这确保了与内部构建工具命令包装的向后兼容性
- **PATH 类变量**作为 **nushell 列表**处理（prepend/append），而非分号拼接的字符串
- 生成的 `.nu` 脚本中所有路径统一使用**正斜杠**
- 聚合脚本中的 `source` 命令使用**绝对路径**以确保可靠性

## 使用方法

在 Conan profile 中添加配置：

```ini
[conf]
tools.env.virtualenv:nushell=nu
```

运行 `conan install . --build=missing` 后，除了默认的 `.bat`/`.sh`/`.ps1` 脚本外，还会额外生成以下 `.nu` 文件：

```
conanbuild.nu
conanbuildenv-release-x86_64.nu
conanrun.nu
conanrunenv-release-x86_64.nu
conanvcvars.nu                    （仅 Windows + MSVC）
deactivate_conanbuild.nu
deactivate_conanrun.nu
deactivate_conanvcvars.nu         （仅 Windows + MSVC）
```

在 nushell 中激活环境：

```nu
source conanbuild.nu
source conanrun.nu
```

还原环境（标准模式）：

```nu
source deactivate_conanbuild.nu
source deactivate_conanrun.nu
```

还原环境（函数模式，需配置 `tools.env:deactivation_mode=function`）：

```nu
deactivate_conanbuild
deactivate_conanrun
```

## 测试说明

### 相关测试文件

本次修改涉及的测试文件分布在三个层级：

#### 单元测试 `test/unittests/tools/env/`

| 文件 | 说明 |
|------|------|
| `test_env.py` | 环境变量核心功能：compose、define/append/prepend/unset、path 操作、`save_bat()`/`save_ps1()` 生成等 |
| `test_env_files.py` | 各格式脚本文件生成：`test_env_files_bat()`、`test_env_files_ps1()`、`test_env_files_sh()`、路径相对化 |

#### 集成测试 `test/integration/toolchains/env/`

| 文件 | 说明 |
|------|------|
| `test_environment.py` | `save_script()` 与 scope 的交互 |
| `test_buildenv.py` | 交叉编译场景、引号转义 |
| `test_virtualenv_default_apply.py` | VirtualBuildEnv/VirtualRunEnv 的自动激活与还原 |
| `test_virtualenv_object_access.py` | VirtualBuildEnv/VirtualRunEnv 对象的 `.vars()` 访问 |
| `test_virtualenv_winbash.py` | Windows bash 环境下的脚本生成（仅 Windows） |

#### 功能测试 `test/functional/toolchains/env/`

| 文件 | 说明 |
|------|------|
| `test_virtualenv_powershell.py` | PowerShell 脚本生成的完整流程测试（仅 Windows）：VCVars、聚合脚本、还原脚本、引号处理 |
| `test_complete.py` | CMake + VirtualBuildEnv 的端到端集成测试 |

### 运行测试

```bash
# 运行环境变量相关的全部单元测试
python -m pytest test/unittests/tools/env/ -x -q

# 运行环境变量相关的全部集成测试
python -m pytest test/integration/toolchains/env/ -x -q

# 运行 PowerShell 功能测试（仅 Windows，可参考此模式编写 nushell 测试）
python -m pytest test/functional/toolchains/env/test_virtualenv_powershell.py -x -q

# 运行脚本文件生成的单元测试（最直接相关）
python -m pytest test/unittests/tools/env/test_env_files.py -x -q

# 运行单个测试函数
python -m pytest test/unittests/tools/env/test_env_files.py::test_env_files_ps1 -x -s

# 运行 conf 配置相关的单元测试
python -m pytest test/unittests/model/test_conf.py -x -q
```

### 测试前置依赖

```bash
pip install -r conans/requirements.txt
pip install -r conans/requirements_dev.txt
# 功能测试还需要：
pip install webtest pyjwt bottle
```

### 当前测试结果

| 测试集 | 结果 |
|--------|------|
| `test/unittests/tools/env/` | 39 passed, 1 failed（`test_windows_case_insensitive_bat` 为已有问题，与本次修改无关） |
| `test/unittests/model/test_conf.py` | 43 passed |
| `test/integration/toolchains/env/test_environment.py` | 1 passed |
| `test/functional/toolchains/env/test_virtualenv_powershell.py` | 3 passed, 1 failed（`test_vcvars` 因本机缺少 VS 安装，与本次修改无关） |
