根据之前的搜索结果，Conan 的环境变量脚本生成核心逻辑分布在以下代码文件中：

### 核心生成逻辑

| 文件路径                                                                     | 作用                                                                                                                           |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `conan/tools/env/environment.py`                                             | **`EnvVars` 类** — 负责将环境变量字典序列化为各种 shell 脚本格式的核心类。`save_script()` 方法决定生成 `.bat` / `.ps1` / `.sh` |
| `conan/tools/env/envvars.py`                                                 | **`EnvVars` 的辅助逻辑**，处理环境变量的增删改查、路径分隔符等                                                                 |
| `conan/tools/env/virtualbuildenv.py`                                         | **`VirtualBuildEnv` 类** — 构建时环境生成器，调用 `EnvVars` 生成 `conanbuildenv` 脚本                                          |
| `conan/tools/env/virtualrunenv.py`                                           | **`VirtualRunEnv` 类** — 运行时环境生成器，生成 `conanrunenv` 脚本                                                             |
| `conan/tools/gnu/autotoolsdeps.py` / `conan/tools/microsoft/visualstudio.py` | 特定工具链的环境变量收集逻辑                                                                                                   |

### 关键源码分析：`environment.py` 中的格式判断

`EnvVars.save_script()` 的核心逻辑大致如下（基于文档和已知行为推断）：

```python
# conan/tools/env/environment.py 中的关键逻辑
def save_script(self, filename):
    _, ext = os.path.splitext(filename)

    if ext == ".bat":
        self.save_bat(filename)      # Windows Batch
    elif ext == ".ps1":
        self.save_ps1(filename)      # PowerShell
    elif ext in (".sh",):
        self.save_sh(filename)       # POSIX Shell (bash/zsh/dash)
    else:
        # 自动检测平台
        if platform.system() == "Windows":
            if self._subsystem == "powershell":
                self.save_ps1(filename)
            else:
                self.save_bat(filename)
        else:
            self.save_sh(filename)
```

### 如果要添加 Nushell / Fish 支持

你需要修改的文件和逻辑：

1. **`conan/tools/env/environment.py`**
    - 在 `save_script()` 方法中添加 `.nu` 和 `.fish` 的分支
    - 实现 `save_nu()` 和 `save_fish()` 方法

2. **Nushell 脚本格式示例**：

    ```python
    def save_nu(self, filename):
        with open(filename, "w") as f:
            f.write("# Nushell environment script\n")
            for name, value in self._env_vars.items():
                # Nushell 语法: $env.NAME = "value"
                f.write(f'$env.{name} = "{value}"\n')
    ```

3. **Fish 脚本格式示例**：

    ```python
    def save_fish(self, filename):
        with open(filename, "w") as f:
            f.write("# Fish shell environment script\n")
            for name, value in self._env_vars.items():
                # Fish 语法: set -gx NAME value
                f.write(f"set -gx {name} {value}\n")
    ```

4. **配置层扩展**（如果需要）
    - `conan/tools/env/virtualbuildenv.py` 和 `virtualrunenv.py` 中可能需要暴露新的配置项（如 `tools.env.virtualenv:shell=nu/fish`）

### 如何找到并修改源码

Conan 2 通常安装在 Python 的 site-packages 中：

```bash
# 找到安装路径
python -c "import conan; print(conan.__file__)"
# 输出类似: /home/user/.local/lib/python3.11/site-packages/conan/__init__.py

# 或者直接查看
pip show conan
```

源码结构：

```
site-packages/conan/
├── tools/
│   └── env/
│       ├── __init__.py
│       ├── environment.py      # <-- 核心修改文件
│       ├── envvars.py
│       ├── virtualbuildenv.py
│       └── virtualrunenv.py
```

### 建议的修改策略

1. **直接修改 `environment.py`**：添加 `save_nu()` / `save_fish()` 方法，并在 `save_script()` 中注册 `.nu` / `.fish` 扩展名
2. **Monkey-patch（如果不想动源码）**：在自己的 `conanfile.py` 中动态替换 `EnvVars.save_script` 方法
3. **提交 PR**：修改后测试通过，可以向 [conan-io/conan](https://github.com/conan-io/conan) 提交 PR。这是一个合理的功能增强，类似的多 Shell 支持在社区已有讨论

如果你需要，我可以帮你写出完整的 `save_nu()` 和 `save_fish()` 实现代码，包括处理 `PATH` 变量的特殊语法（Nushell 使用 `$env.PATH = ($env.PATH | prepend "/new/path")`，Fish 使用 `set -gxp PATH /new/path`）。
