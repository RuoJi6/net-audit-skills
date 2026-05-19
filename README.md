## 审计skills系列

- [java-audit-skills](https://github.com/RuoJi6/java-audit-skills) - Java 代码审计 Claude Skills 集合

# .NET Audit Skills

专注于 .NET / ASP.NET 代码审计的 Claude Skills 集合，提供自动化源码与反编译产物分析、路由提取、参数映射等功能，辅助安全研究人员和开发者进行 ASP.NET Web 应用的安全审计工作。

## 功能特性

- **自动路由识别**：自动识别 ASP.NET Web 项目中的 HTTP 路由结构
- **鉴权机制审计**：梳理登录校验、权限校验、角色/Policy、资源级授权与越权风险
- **多框架支持**：支持 ASP.NET MVC 5 及以下、ASP.NET Core (.NET 5/6/7+)、ASP.NET Web Forms、Web API 2、最小 API 等主流框架
- **参数结构解析**：提取 Query / Form / Body / Path / Header / Cookie / RouteValue 等各类参数
- **反编译集成**：集成 .NET 反编译器（ILSpy / ilspycmd），支持分析闭源 `.dll` / `.exe` 程序集
- **Burp Suite 集成**：生成可直接用于 Burp Suite Repeater 的请求模板
- **接口文档生成**：为无 API 文档的项目生成接口清单
- **完整输出保证**：必须输出所有发现的接口，不允许省略

> 当前已发布 **net-route-mapper** 与 **net-auth-audit**；SQL 注入审计、文件上传 / 读取审计、XXE 审计、组件漏洞扫描及全链路审计流水线等 skill 正在规划中（详见末尾 TODO）。

## 前置要求

- **.NET 运行环境**：需安装 .NET SDK 或 .NET Runtime（`dotnet --info` 可用即可，用于运行反编译相关命令行工具）
- **ILSpy 反编译器**：有两种获取方式：
  1. 已有本地 `ilspycmd` / ILSpy GUI，直接指定使用
  2. 通过 `dotnet tool install -g ilspycmd` 安装命令行版本
- 闭源项目同时建议准备 `dnSpyEx` / `dotPeek` 作为辅助交叉验证工具

详细反编译策略参见 `skills/net-route-mapper/references/DECOMPILE_STRATEGY.md`

## 目录结构

```
net-audit-skills/
├── README.md                   # 项目说明文档
└── skills/                     # Skills 集合目录
    ├── net-shared/             # .NET 审计共享标准
    │   └── SEVERITY_RATING.md  # 漏洞编号与 R/I/C 评分标准
    ├── net-route-mapper/       # ASP.NET 路由与参数映射工具
    │   ├── SKILL.md
    │   └── references/
    │       ├── ASPNET_CORE.md         # ASP.NET Core 路由识别参考
    │       ├── ASPNET_MVC5.md         # ASP.NET MVC 5 路由识别参考
    │       ├── ASPNET_WEBFORMS.md     # ASP.NET Web Forms 路由识别参考
    │       ├── DECOMPILE_STRATEGY.md  # 共享二进制预处理 / 反编译策略
    │       └── OUTPUT_TEMPLATE.md     # 输出模板
    └── net-auth-audit/         # ASP.NET 鉴权机制审计工具
        ├── SKILL.md
        └── references/
            ├── AUTH_PATTERNS.md       # .NET 鉴权入口识别参考
            ├── EVAL_CASES.md          # 触发边界与失败案例评测样例
            ├── IDOR_CHECKLIST.md      # 水平越权 / 资源归属检查清单
            ├── OUTPUT_TEMPLATE_MAIN.md
            ├── OUTPUT_TEMPLATE_MAPPING.md
            └── OUTPUT_TEMPLATE_README.md
```

## 可用 Skills

| Skill              | 说明                                       |
| ------------------ | ------------------------------------------ |
| net-route-mapper   | ASP.NET 源码路由与参数映射分析工具         |
| net-auth-audit     | ASP.NET 鉴权机制、权限校验与越权风险审计工具 |

## 安装与使用

### 1. 准备 ILSpy 反编译器（仅闭源项目需要）

```bash
# 通过 dotnet tool 安装命令行版本
dotnet tool install -g ilspycmd

# 验证
ilspycmd --help
```

### 2. 配置 Skills

将 `skills/` 目录下的内容复制到 Claude Code 的 skills 配置目录中。

### 3. 使用 Skill

在 Claude Code 中调用 skill：

```
/net-route-mapper /path/to/project
/net-auth-audit /path/to/project
```

**参数说明：**

- `/path/to/project` - .NET 项目根目录，包含源码（`.cs` / `.aspx`）或编译产物（`bin/*.dll`）
- `net-auth-audit` 可复用 `net-route-mapper` 输出和 `{project}_audit/decompiled/manifest.json` 共享反编译产物
- `net-auth-audit` 输出固定保存到 `{project}_audit/auth_audit/`，包含 `README.md`、`auth_mapping.md`、`auth_findings.md` 和 `modules/`

**项目路径要求：**

- 源码项目：包含 `*.csproj` 及 `.cs` / `.aspx` / `Program.cs` / `Startup.cs` / `App_Start/RouteConfig.cs` 等
- 编译项目：包含 `bin/` 目录下的 `.dll` / `.exe`
- 支持同时存在源码和编译文件的情况，优先使用源码

## 支持的项目类型

| 类型                      | 入口标识                                 | 路由载体                                                       |
| ------------------------- | ---------------------------------------- | -------------------------------------------------------------- |
| ASP.NET Web Forms         | `.aspx` / `.ashx` / `.asmx`              | 物理文件路径 + `Global.asax` 中 `MapPageRoute`                 |
| ASP.NET MVC 5 及以下      | `App_Start/RouteConfig.cs` + `Global.asax` | `RouteCollection.MapRoute` + Controller/Action 约定           |
| Web API 2 (.NET Framework) | `App_Start/WebApiConfig.cs`             | `config.Routes.MapHttpRoute` + `[RoutePrefix]`/`[Route]`       |
| ASP.NET Core (.NET 5/6/7+) | `Program.cs` / `Startup.cs`             | `MapControllerRoute` / `MapControllers` / 属性路由 `[Route]`/`[HttpGet]` |
| 最小 API (.NET 6+)        | `Program.cs`                             | `app.MapGet` / `app.MapPost` / ...                             |

## 最佳实践

1. 优先使用源码，仅在必要时使用反编译
2. 记录每个路由的源文件位置（含命名空间.类名）便于追溯
3. 输出格式统一，便于后续处理
4. 遇到无法解析的配置时记录并跳过
5. 闭源 `.dll` 必须先反编译再分析，禁止凭文件名猜测路由

## 贡献

欢迎提交 Issue 和 Pull Request 来完善项目功能。

## 许可证

本项目仅供学习和研究使用。

## TODO 待办列表

- [ ] **net-sql-audit**：ASP.NET SQL 注入漏洞审计工具（ADO.NET / Dapper / EF Core）
- [ ] **net-file-upload-audit**：ASP.NET 文件上传漏洞审计工具
- [ ] **net-file-read-audit**：ASP.NET 文件读取漏洞审计工具
- [ ] **net-xxe-audit**：ASP.NET XXE 漏洞审计工具（XmlDocument / XmlReader / DataSet）
- [ ] **net-vuln-scanner**：.NET 组件版本漏洞检测工具（基于 NuGet `packages.config` / `*.csproj` / `*.deps.json`）
- [ ] **net-route-tracer**：ASP.NET 路由多层级调用链追踪工具
- [ ] **net-audit-pipeline**：.NET 全链路自动化安全审计流水线（agent teams 编排）

## 代码审计培训推荐(备注申请：弱鸡推荐)

![代码审计培训推荐](assets/WechatIMG3380.jpg)

## 相关链接

- [ILSpy](https://github.com/icsharpcode/ILSpy) - .NET 反编译器
- [Claude Code](https://claude.ai/claude-code) - Claude CLI 工具
