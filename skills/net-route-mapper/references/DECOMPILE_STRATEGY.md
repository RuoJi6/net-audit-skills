# .NET 反编译策略

适用于仅有 `bin/*.dll` 而无 `.cs` 源码的闭源审计场景，或需要还原 Razor 视图、`Global.asax` 隐藏代码等。

> **完整工作流**：先按 §8 识别是否混淆 → 若混淆则按 §9 用 `de4dot` 脱壳 → 再按 §1~§4 用 `ilspycmd` 反编译落盘 → 按 §3 全局搜索定位路由。

## 1. 反编译工具：ilspycmd（统一使用）

本 skill **统一使用 `ilspycmd`** 作为反编译工具，原因：

- 跨平台（Windows / macOS / Linux），通过 `dotnet tool` 安装即可
- 可批量、可脚本化，将整个 `bin/` 目录一次性导出为 C# 项目
- 输出落盘后便于 `grep` / `rg` 静态分析，符合下游 skill 消费方式

### 1.1 安装

```bash
# 全局安装（需要 .NET SDK 6.0+）
dotnet tool install -g ilspycmd

# 升级
dotnet tool update -g ilspycmd

# 验证
ilspycmd --version
```

> 若仅用于单次反编译且不能联网，可用本地包：`dotnet tool install -g ilspycmd --add-source ./nupkgs`。

### 1.2 常用参数（必须掌握）

| 参数 | 含义 |
| --- | --- |
| `-o <dir>` | 输出目录 |
| `-p` | 以 C# 项目（`.csproj` + 完整目录结构）形式导出，强烈推荐 |
| `-r` / `--referencepath <dir>` | 额外引用程序集搜索目录（解决依赖找不到） |
| `--nested-directories` | 按命名空间生成嵌套目录 |
| `--no-dead-code` / `--no-dead-stores` | 删除死代码 / 死存储，提升可读性 |
| `-lv <CSharp7_3\|Latest>` | 目标 C# 语言版本 |

### 1.3 单文件 / 批量反编译

```bash
# 单个 dll → C# 项目
ilspycmd bin/WebApplication1.dll -p \
  -o {project}_audit/decompiled/WebApplication1/

# 批量：bin 目录下所有项目 dll（排除微软自带库）
mkdir -p {project}_audit/decompiled
for f in bin/*.dll; do
  name=$(basename "$f" .dll)
  case "$name" in
    System.*|Microsoft.*|Mono.*|netstandard|mscorlib) continue ;;
  esac
  ilspycmd "$f" -p \
    --referencepath bin \
    -o "{project}_audit/decompiled/$name/"
done
```

### 1.4 备选工具（仅在 ilspycmd 不可用时）

| 工具 | 适用 | 备注 |
| --- | --- | --- |
| **dnSpy** | Windows / Wine | v6.1.8 GUI，仅在需要交互式调试时使用 |
| **dotPeek** | Windows | JetBrains，可作为符号服务器 |
| **ILSpy GUI** | Windows | `ilspycmd` 同源 |

> **本 skill 默认输出步骤一律按 `ilspycmd` 描述。** 备选工具仅作应急。

## 2. 加载与浏览策略

1. 把项目 `bin/` 目录下**所有** dll 用 §1.3 的批量脚本一次性反编译落盘到 `{project}_audit/decompiled/`
2. 排除 .NET 自带库
   - 命名空间以 `System.*` / `Microsoft.*` / `Mono.*` 开头的程序集通常无需关注
   - 但 `Microsoft.AspNetCore.*` 可能含项目自身扩展，按需进入
3. 优先反编译入口程序集（与 web.config 中 `<compilation>` 或 .aspx 的 `Inherits` 命名空间一致）
4. 反编译完成后通过 `rg` / `grep -R` 在 `decompiled/` 目录中执行 §3 的全局搜索

## 3. 路由定位流程

在 `{project}_audit/decompiled/` 目录上执行 **`rg`/`grep -R`** 全局搜索（按以下顺序）：

1. `RegisterRoutes` —— ASP.NET MVC 5 / Web API 2 路由注册主入口
2. `MapPageRoute` —— Web Forms 友好 URL
3. `MapControllerRoute` / `MapControllers` / `MapGet` / `MapPost` —— ASP.NET Core
4. `Application_Start` —— `Global.asax` 隐藏代码（启动钩子）
5. `namespace ` + `.Controllers` —— 定位控制层（过滤 `System.*`）
6. `: Controller` / `: ApiController` / `: ControllerBase` —— 定位控制器基类
7. `IHttpHandler` —— 定位 .ashx 处理器
8. `[WebMethod]` —— 定位 .asmx 方法

> 仅有 `Global.asax` 文件时：先反编译 `Global.asax` 对应 dll 的 `Application_Start`，再顺藤摸瓜找到 `RouteConfig` / `WebApiConfig` 类。

## 4. 落盘策略

把反编译产物以 `.cs` 文件落盘，便于后续 `grep` / 静态分析：

```bash
ilspycmd bin/MyApp.dll -p \
  --referencepath bin \
  -o {project}_audit/decompiled/MyApp/
```

- 必须使用 `-p` 以项目形式导出（而非单文件），保留命名空间目录结构
- `--referencepath bin` 用于解决跨程序集依赖类型解析失败
- 统一输出到 `{project}_audit/decompiled/{AssemblyName}/`，供下游 net-route-tracer / net-sql-audit / net-auth-audit 直接消费

## 5. 与 .aspx 的对应

`.aspx` 顶部 `Inherits="X.Y.Z.ClassName"`：
- 命名空间前缀 = dll 文件名（去掉 `.dll`）
  - 例：`WebApplication1.Contact` → `bin/WebApplication1.dll`
- 类名 = dll 中对应的类
- 所有 `Page_Load` / 控件事件方法都在该类中
- 若类名带子命名空间（如 `JHSoft.Web.WorkFlat.DBModules`），需在 dll 中按完整路径定位

## 6. 反编译陷阱

| 陷阱 | 处理 |
| --- | --- |
| 异步 / 迭代器方法被改写为状态机 | 关注 `MoveNext` / `<>c__DisplayClass` 中的字段引用 |
| 字符串拼接被编译器优化为 `String.Concat` 数组 | 重点看常量字符串数组 |
| 顶级语句（.NET 6+）→ 隐式 `Program.<Main>$` | 直接进 `Program.<Main>$` 阅读路由注册代码 |
| `[CompilerGenerated]` 类 | 通常忽略，但 lambda / 闭包可能含真实业务逻辑 |
| 混淆的程序集（ConfuserEx / Eazfuscator） | 先用 de4dot 脱壳后再反编译 |

## 7. 反编译产物归档建议

```
{project}_audit/decompiled/
├── WebApplication1/
│   ├── Controllers/
│   │   ├── HomeController.cs
│   │   └── AccountController.cs
│   ├── App_Start/
│   │   └── RouteConfig.cs
│   ├── Default.aspx.cs
│   └── ...
└── JHSoft.Web.WorkFlat/
    └── DBModules.cs
```

供下游 net-route-tracer / net-sql-audit / net-auth-audit 等 skill 直接消费。

---

## 8. 混淆识别

在反编译之前必须先判断目标 dll 是否被混淆。一旦反编译产物出现下列任一特征，立即停止常规反编译，转入 §9 用 `de4dot` 脱壳后重做。

### 8.1 典型混淆特征

```csharp
// 特征 1：成员名被替换为不可见 Unicode 字符（\u0001 / \u0002 / \u0003 …）
private string \u0001;
private void \u0002(string \u0003)
{
    if (this.\u0001 == \u0003)
}

// 特征 2：类 / 命名空间名为单字符或纯符号（A、B、a、b、1、I 等无意义短名）
// 特征 3：字符串常量被加密、运行时通过解密函数还原
// 特征 4：方法体大量 try/catch + goto + 无意义跳转（控制流平坦化）
// 特征 5：dll 元数据中出现 "Confused by ConfuserEx" / "Eazfuscator.NET" 等水印
```

### 8.2 快速排查命令

```bash
# 在反编译输出目录搜索 \uXXXX 风格成员名
grep -rE '\\u00[0-9a-fA-F]{2}' decompiled/ | head

# 查看 dll 元数据中的混淆器水印
strings bin/YourApp.dll | grep -iE 'confuser|eazfuscator|smartassembly|dotfuscator'
```

## 9. 去混淆：de4dot

[de4dot](https://github.com/kant2002/de4dot) 是 .NET 通用脱壳器，可将打包/混淆后的程序集尽可能还原为接近原始的形态。**字符串解密、控制流还原、代理方法移除、资源解密、虚拟化代码反虚拟化** 等都能自动处理；但 **符号重命名无法 100% 还原**（混淆器通常不会保留原名），de4dot 会重命名为可读符号（如 `Class0`、`method1`）。

**统一使用社区活跃维护分支 [kant2002/de4dot](https://github.com/kant2002/de4dot)**（最新 v3.2.0，2025-08；原 `0xd4d/de4dot`、`ViRb3/de4dot-cex` 已停更，对新版混淆器支持不足）。

### 9.1 支持的混淆器（截至 v3.2.0）

Agile.NET (CliSecure) / Babel.NET / CodeFort / CodeVeil / CodeWall / CryptoObfuscator / DeepSea / Dotfuscator / .NET Reactor / **Eazfuscator.NET** / Goliath.NET / ILProtector / MaxtoCode / MPRESS / Rummage / Skater.NET / **SmartAssembly** / Spices.Net / Xenocode。

> **ConfuserEx / Confuser** 不在官方列表，但 kant2002 分支合并了 `mobile46` 的现代混淆器支持补丁；实际可处理大部分 ConfuserEx 变种，遇到失败再切换 dnSpy 动态调试。

### 9.2 安装

`kant2002/de4dot` 同时编译 `net48` 与现代 .NET（v3.2.0 release 含 net48 + net8.0-winx64）。**macOS 用户用 mono 跑 net48 二进制最快**，详见 §9.2.1。

#### 9.2.1 macOS（推荐：mono + net48 release，1 分钟搞定）

```bash
# 1) 安装 mono
brew install mono
# 若运行时报 System.Drawing 缺失，再装：
#   brew install mono-libgdiplus

# 2) 下载 net48 release（约 1 MB）
curl -L -o /tmp/de4dot-net48.zip \
  https://github.com/kant2002/de4dot/releases/download/v3.2.0/de4dot-net48.zip
mkdir -p ~/tools/de4dot
unzip /tmp/de4dot-net48.zip -d ~/tools/de4dot

# 3) 写 alias 到 ~/.zshrc（或 ~/.bashrc）
echo "alias de4dot='mono $HOME/tools/de4dot/de4dot.exe'" >> ~/.zshrc
source ~/.zshrc

# 4) 验证
de4dot -h | head
```

#### 9.2.2 Linux

同 macOS 思路，将 `brew install mono` 替换为：

```bash
sudo apt-get install -y mono-complete       # Debian / Ubuntu
# 或：sudo dnf install -y mono-complete       # Fedora
```

#### 9.2.3 Windows

直接下载 [release](https://github.com/kant2002/de4dot/releases) 的 `de4dot-net8.0-winx64.zip`（自包含，无需装 .NET）或 `de4dot-net48.zip`（需 .NET Framework 4.8），解压后将目录加入 PATH。

### 9.3 检测混淆器（先跑，确认可处理）

```bash
# 单文件检测
de4dot -d file1.dll

# 整目录递归检测，仅显示已识别为某混淆器的文件
de4dot -d -r ./bin -ru
```

输出形如 `Detected SmartAssembly` / `Unknown obfuscator`。若全部为 `Unknown` 说明未混淆或自研壳，直接进入 ilspycmd 反编译；若识别成功才进入下一步。

### 9.4 单文件去混淆

```bash
# 默认输出同目录下 file1-cleaned.dll
de4dot file1.dll

# -o 显式指定输出文件名
de4dot source.dll -o source-clean.dll
```

### 9.5 整目录递归去混淆

```bash
# -r 递归输入目录   -ru 忽略未识别（非混淆/非程序集）文件   -ro 输出目录
de4dot -r ./bin -ru -ro ./bin_clean
```

> ⚠️ **多 dll 必须一次性处理**：当一个 dll 引用另一个 dll 中的类时，单独处理会让重命名后的引用断链。务必把所有相关 dll 在**同一条 `-r/-ro` 命令**内处理。

### 9.6 常用场景参数

| 场景 | 命令 |
| --- | --- |
| 强制按某混淆器处理（自动检测失败时） | `de4dot file.dll -p sa`（sa=SmartAssembly，其它见 `de4dot --help`） |
| 不重命名符号（保留原名，避免反射断链 / WPF/XAML / Silverlight） | `de4dot --dont-rename file.dll` |
| 保留元数据 token（动态调试 / 二次脱壳） | `de4dot --preserve-tokens file.dll` |
| 保留类型不删除（保留混淆器添加的类，便于排查） | `de4dot --keep-types --preserve-tokens file.dll` |
| 动态解密字符串（已知解密方法 token） | `de4dot file.dll --strtyp delegate --strtok 06012345` |

### 9.7 批量 Python 脚本（按子目录分散输出，便于审计追溯）

把下面脚本保存为 `de4dot_batch.py`，对 `bin/` 直接 `python3 de4dot_batch.py -i bin -o bin_clean -r` 跑批量：

```python
#!/usr/bin/env python3
"""批量 de4dot 去混淆。会跳过常见第三方库（System.* / Microsoft.* 等）。"""
import argparse
import subprocess
import time
from pathlib import Path

EXCLUDE_PATTERNS = [
    "System.", "Microsoft.", "Newtonsoft.", "EntityFramework",
    "Oracle.", "MySql.", "NLog.", "Quartz.", "RestSharp",
    "StackExchange.", "Thinktecture.", "BouncyCastle",
]


def main() -> None:
    parser = argparse.ArgumentParser(description="批量 de4dot 去混淆")
    parser.add_argument("-i", "--input", default=".", help="输入目录（默认当前目录）")
    parser.add_argument("-o", "--output", default="deobfuscated", help="输出目录")
    parser.add_argument("-r", "--recursive", action="store_true", help="递归子目录")
    args = parser.parse_args()

    # 检查 de4dot
    try:
        r = subprocess.run(["de4dot", "--help"], capture_output=True, text=True, timeout=5)
        if r.returncode != 0:
            raise RuntimeError("de4dot 不可用")
    except Exception as e:
        print(f"✗ de4dot 不可用: {e}")
        return

    input_dir = Path(args.input).resolve()
    output_dir = Path(args.output).resolve()
    output_dir.mkdir(parents=True, exist_ok=True)

    glob = "**/*" if args.recursive else "*"
    files = [f for f in list(input_dir.glob(f"{glob}.dll")) + list(input_dir.glob(f"{glob}.exe"))
             if not any(p in f.name for p in EXCLUDE_PATTERNS)]
    if not files:
        print("✗ 未找到需处理的 dll/exe")
        return

    print(f"找到 {len(files)} 个待处理文件")
    ok = fail = 0
    t0 = time.time()
    for i, f in enumerate(files, 1):
        rel = f.relative_to(input_dir)
        out_subdir = output_dir / rel.parent / f.stem
        out_subdir.mkdir(parents=True, exist_ok=True)
        out_file = out_subdir / f.name
        print(f"[{i}/{len(files)}] {f.name}")
        try:
            r = subprocess.run(["de4dot", str(f), "-o", str(out_file)],
                               capture_output=True, text=True, timeout=300)
            if r.returncode == 0:
                ok += 1
            else:
                fail += 1
                print(f"  ✗ {r.stderr[:120]}")
        except subprocess.TimeoutExpired:
            fail += 1
            print("  ✗ 超时")

    print(f"完成: ok={ok} fail={fail} 耗时={time.time() - t0:.1f}s 输出={output_dir}")


if __name__ == "__main__":
    main()
```

### 9.8 与 ilspycmd 的串联

```bash
# 1) 去混淆：bin/ → bin_clean/
python3 de4dot_batch.py -i bin -o bin_clean -r

# 2) 用清洗后的 dll 反编译（参考 §1.4 批量脚本，把 bin/ 替换为 bin_clean/）
ilspycmd -p --referencepath bin_clean -o {project}_audit/decompiled/YourApp/ bin_clean/YourApp.dll
```

### 9.9 去混淆陷阱

- **强名称失效**：脱壳后 dll 签名失效，无法在生产环境替换运行；本审计流程仅用于静态分析，可忽略。
- **反射调用断链**：de4dot 默认会重命名符号，若代码用 `Type.GetType("X.Y")` / `Assembly.GetType` 反射加载，重命名后会找不到原类。表现为「脱壳后程序运行报 TypeLoadException，但静态读代码没问题」——审计场景仅做静态读，可忽略；若要动态运行验证，加 `--dont-rename`。
- **WPF / Silverlight / XAML**：XAML 引用未脱敏的类名，重命名会断 UI 绑定。如目标是 WPF 项目，**务必加 `--dont-rename`**。
- **多 dll 必须一次脱壳**：见 §9.5 警告，分批处理会让程序集间引用错位。
- **未支持的混淆器**：极少数自研壳 / 新版 ConfuserEx 变种 de4dot 输出 `Unknown obfuscator` 时会原样输出；此时切 dnSpy 动态调试或人工反汇编。
- **必须先去混淆再反编译**：直接用 ilspycmd 输出的 `\u0001` 类成员无法被 grep / 路由定位关键词命中，会漏掉接口。
- **混淆器自身被加壳的场景**：先用 PEiD/Detect It Easy 检查是否还套了原生壳（UPX/MPRESS），有则先 `upx -d` 脱原生壳再用 de4dot 处理 .NET 层。

## 10. 参考链接

- ICSharpCode/ILSpy（ilspycmd）：https://github.com/icsharpcode/ILSpy
- de4dot（活跃维护分支，推荐）：https://github.com/kant2002/de4dot
- 实战参考：https://forum.butian.net/share/4655
