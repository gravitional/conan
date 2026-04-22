# 添加 Nushell 支持的开发经验

## 背景

为 Conan 添加 Nushell (`.nu`) 环境脚本生成支持，与已有的 `.bat` / `.ps1` / `.sh` 保持一致。

---

## 核心改动文件

| 文件                                         | 改动                                                                                                |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `conan/tools/env/environment.py`             | 新增 `save_nu()`、`_nu_deactivate_contents()`，修改 `save_script()` 和 `environment_wrap_command()` |
| `test/unittests/tools/env/test_env_files.py` | 新增 `test_env_files_nu` 集成测试                                                                   |

---

## 技术难点与解决方案

### 1. Nushell 的 `source` 是编译时指令

**问题**：
Nushell 的 `source file.nu` 在脚本**编译时**就需要 `file.nu` 存在，而不是运行时。这与 bash 的 `. ./script.sh` 完全不同。

这意味着不能在一个脚本里先 `save deactivate_test.nu` 再 `source deactivate_test.nu`。在 bash/ps1 测试中常用的：

```bash
. ./test.sh && ./display.sh && . ./deactivate_test.sh && ./display.sh
```

在 Nushell 中完全不可行。

**解决方案**：

- 分两步测试：
    1. `nu test.nu` —— 运行激活脚本，生成 `deactivate_test.nu`
    2. `nu deactivate_verify.nu` —— 在新进程中先设置激活后的值，再 `source deactivate_test.nu`，验证恢复

### 2. 字符串插值与占位符展开

**问题**：
PowerShell 的双引号字符串会自动展开 `$env:VAR`，bash 也会展开 `$VAR`。
但 Nushell 的普通双引号 `"..."` **不会**自动展开 `$env.VAR`。

例如 `get_str("$env.{name}", ...)` 返回 `"$env.MyVar2 MyValue2"`：

- 在 ps1 中直接赋值：`$env:MyVar2 = "$env:MyVar2 MyValue2"` → PowerShell 自动展开为 `old MyValue2`
- 在 Nushell 中直接赋值：`$env.MyVar2 = "$env.MyVar2 MyValue2"` → 值就是字面量 `$env.MyVar2 MyValue2`

**解决方案**：
使用 Nushell 的插值语法 `$"..."`，并将占位符替换为带默认值保护的表达式：

```nu
if ("MyVar2" in ($env | columns)) {
    $env.MyVar2 = $"($env.MyVar2? | default '') MyValue2"
} else {
    $env.MyVar2 = "MyValue2"
}
```

### 3. 自定义命令修改环境变量必须使用 `def --env`

**问题**：
Nushell 中普通 `def` 定义的函数有自己的作用域，无法修改调用者的环境变量。

**解决方案**：
function 模式的 deactivate 必须使用 `def --env`：

```nu
def --env deactivate_xxx [] {
    load-env { ($v): $val }
    hide-env $old_var
}
```

### 4. 动态列名操作

**问题**：
Nushell 不支持 `$env.($varname) = $in` 这种动态列名赋值。

**解决方案**：
使用 `load-env` 配合动态键名：

```nu
let v = "MYVAR"
load-env { ($v): $val }
```

### 5. 文件编码必须是无 BOM 的 UTF-8

**问题**：
Nushell 解析器会把 BOM（`EF BB BF`）当成普通字符，导致 `$env.VAR` 被解析成 `.VAR`，报错 `assignment_requires_variable`。

**解决方案**：
Python 写入文件时使用 `encoding="utf-8"`（Python 3 的 `utf-8` 默认不带 BOM）。

### 6. `environment_wrap_command` 的扩展

添加 `.nu` 支持时，需要在 `environment_wrap_command` 中：

- 新增 `nus` 列表收集 `.nu` 脚本
- 在 `else` 分支（无扩展名时）探测 `.nu`
- 添加 `elif nus:` 包装逻辑：`nu -c "source ... ; cmd"`

---

## 如何测试

### 环境要求

需要安装 Nushell，并确保 `nu` 命令在 `PATH` 中可用：

```powershell
# 检查 nu 是否安装
nu --version
```

### 运行 Nushell 相关测试

```bash
# 只运行 Nushell 测试
python -m pytest test\unittests\tools\env\test_env_files.py::test_env_files_nu -xvs

# 运行所有环境文件测试（包括 bat/ps1/sh/nu）
python -m pytest test\unittests\tools\env\test_env_files.py -xvs
```

### 测试文件结构

| 文件                   | 作用                                                             |
| ---------------------- | ---------------------------------------------------------------- |
| `test.nu`              | Conan 生成的激活脚本，包含环境变量设置 + deactivate 文件生成逻辑 |
| `display.nu`           | 测试辅助脚本，打印所有被测环境变量的当前值                       |
| `deactivate_test.nu`   | 由 `test.nu` 动态生成的反激活脚本                                |
| `deactivate_verify.nu` | 测试辅助脚本，先设置激活后的值，再 source deactivate，验证恢复   |

### 测试流程（分步原因）

由于 Nushell 的 `source` 是**编译时指令**，不能像 bash 那样在同一条命令里先激活再反激活：

```bash
# bash 可以这样（同一进程，顺序执行）
. ./test.sh && ./display.sh && . ./deactivate_test.sh && ./display.sh

# Nushell 不行！因为 source 在编译时就需要文件存在
# 以下命令会在编译阶段就报错：File not found: deactivate_test.nu
nu -c "source test.nu; source display.nu; source deactivate_test.nu; source display.nu"
```

因此测试拆为三步：

**Step 1 — 生成 deactivate 文件**

```bash
nu test.nu
```

运行激活脚本，副作用是生成 `deactivate_test.nu`。此进程结束后环境变量随之消失，但文件已落盘。

**Step 2 — 验证 activate**

```bash
nu -c "source test.nu; source display.nu"
```

在新进程中 source 激活脚本 + 打印变量，验证各环境变量被正确设置。

**Step 3 — 验证 deactivate**

```bash
nu deactivate_verify.nu
```

`deactivate_verify.nu` 的内容示例：

```nu
$env.MyVar = "MyValue"           # 模拟 activate 后的状态
$env.MyVar1 = "MyValue1"
...
source deactivate_test.nu        # 执行反激活
source display.nu                # 打印恢复后的值
```

验证各变量恢复到了 `prevenv` 中定义的初始值。

### 手动调试脚本

如果想本地手动验证生成的 `.nu` 脚本：

```python
from conan.tools.env import Environment
from conan.test.utils.mocks import ConanFileMock

env = Environment()
env.define("MY_VAR", "hello")
env.append("MY_PATH", "/some/path")

e = env.vars(ConanFileMock())
e.save_nu("conanenv.nu")
```

然后在 Nushell 中：

```bash
# 查看生成的脚本内容
cat conanenv.nu

# 激活
source conanenv.nu

# 查看变量
$env.MY_VAR

# 反激活（需先运行 activate 生成 deactivate 文件）
source deactivate_conanenv.nu
```

---

## 测试设计

由于 Nushell 的 `source` 编译时特性，无法单条命令完成 activate → display → deactivate → display 的全流程验证。

测试分三步：

```python
def test_env_files_nu(env, prevenv):
    # Step 1: 运行激活脚本，生成 deactivate 文件
    subprocess.run("nu test.nu", env=prevenv, shell=True, check=True)

    # Step 2: 验证 activate 后的环境
    cmd_activate = 'nu -c "source test.nu; source display.nu"'
    # 断言各变量值正确...

    # Step 3: 验证 deactivate 恢复
    # 先构造一个脚本：设置激活后的值 → source deactivate → 打印
    deactivate_verify = """
        $env.MyVar = "MyValue"
        ...
        source deactivate_test.nu
        source display.nu
    """
    # 断言各变量恢复为旧值...
```

---

## Nushell 语法速查

| 需求                   | Nushell 语法                      |
| ---------------------- | --------------------------------- |
| 设置环境变量           | `$env.NAME = "value"`             |
| 删除环境变量           | `hide-env NAME`                   |
| 检查变量存在           | `"NAME" in ($env \| columns)`     |
| 安全读取（可能不存在） | `$env.NAME? \| default ''`        |
| 字符串插值             | `$"hello ($name)"`                |
| 动态键名赋值           | `load-env { ($v): $val }`         |
| 自定义命令改环境       | `def --env name [] { ... }`       |
| 保存文件               | `"text" \| save file.nu`          |
| 追加文件               | `"text" \| save --append file.nu` |

---

## 关键教训

1. **不要假设所有 shell 的 `source`/`.` 行为相同**。Nushell 的 `source` 是编译时的，这是最大的陷阱。
2. **占位符展开策略因 shell 而异**。ps1/sh 靠运行时字符串展开，Nushell 需要显式插值 `$"..."`。
3. **Nushell 的作用域更严格**。`def` vs `def --env`、`let` vs `$env.` 有本质区别。
4. **BOM 问题**。Windows 上很多编辑器/API 默认带 BOM，Nushell 会因此解析失败。
5. **测试要分进程**。由于环境变量不跨进程，Nushell 的测试需要分步调用 `nu`。
