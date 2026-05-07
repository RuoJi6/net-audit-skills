---
name: net-route-mapper
description: ASP.NET 源码路由与参数映射分析工具。从 .NET 项目源码或反编译产物中提取**所有** HTTP 路由（含控制器/Action/Web Forms 页面）和参数结构，并自动保存为 MD 文档。适用于：(1) 无 API 文档的 .NET 项目完整接口梳理，(2) 下游漏洞审计 Skill 的路由数据源，(3) 闭源 dll 反编译后的端点完整分析。支持 ASP.NET MVC 5 及以下、ASP.NET Core (.NET 5/6/7+)、ASP.NET Web Forms 三大类框架。**必须输出所有接口，不省略任何内容**。
---

# ASP.NET Source Route & Parameter Mapper

从 ASP.NET Web 项目源码（含闭源 dll 反编译产物）中**提取**所有 HTTP 路由与请求参数结构，为下游漏洞审计 Skill 提供完整的路由数据。**不进行安全漏洞评估、代码质量分析或任何路由提取范围之外的内容输出。**

---

## ⚠️ 核心要求：完整输出

**此技能必须输出所有发现的路由，不允许省略。**

- ✅ 每个路由都要有完整的参数分析（Query / Form / Body / Path / Header / Cookie / RouteValue）
- ✅ 输出路由总数和清单供核对
- ❌ 禁止使用 "..."、"等"、"其他" 省略
- ❌ 禁止只输出"关键接口"或"重要接口"
- ❌ 禁止因为数量大而省略

---

## 📌 适用场景

- 无 API 文档的 ASP.NET 项目接口梳理
- 对闭源 dll 反编译后还原可访问端点
- 为 net-auth-audit / net-sql-audit 等下游 skill 提供输入

## 🧩 支持的项目类型

| 类型 | 入口标识 | 路由载体 |
| --- | --- | --- |
| ASP.NET Web Forms | `.aspx` / `.ashx` / `.asmx` | 物理文件路径 + `Global.asax` 中 `MapPageRoute` |
| ASP.NET MVC 5 及以下 | `App_Start/RouteConfig.cs` + `Global.asax` | `RouteCollection.MapRoute` + Controller/Action 约定 |
| ASP.NET Core (.NET 5/6/7+) | `Program.cs` / `Startup.cs` | `MapControllerRoute` / `MapControllers` / 属性路由 `[Route]`/`[HttpGet]` |
| Web API 2 (.NET Framework) | `App_Start/WebApiConfig.cs` | `config.Routes.MapHttpRoute` + `[RoutePrefix]`/`[Route]` |
| 最小 API (.NET 6+) | `Program.cs` | `app.MapGet/MapPost/...` |

---

## 0. 项目类型识别（必须第一步）

执行任何路由提取前，必须先按以下流程判定项目类型：

### 0.1 看入口文件后缀

- 存在大量 `.aspx`/`.asax`/`.ashx`/`.asmx` 物理文件 → **可能是 Web Forms**（也可能在 MVC 中混用）
- 存在 `App_Start/RouteConfig.cs` → **ASP.NET MVC 5 或更早**
- 存在 `App_Start/WebApiConfig.cs` → **Web API 2**
- 存在 `Program.cs` 中调用 `WebApplication.CreateBuilder` → **ASP.NET Core / 最小 API**
- 仅有 `bin/` 目录下的 `.dll` 而无 `.cs` 源码 → **闭源项目，必须反编译**（见 `references/DECOMPILE_STRATEGY.md`）

### 0.2 看 .aspx 顶部声明

每个 `.aspx` 顶部的 `<%@ Page %>` 指令是定位后端代码的关键：

```aspx
<%@ Page Language="C#" AutoEventWireup="true"
         CodeBehind="DBModules.aspx.cs"
         Inherits="JHSoft.Web.WorkFlat.DBModules" %>
```

提取规则：
- `Inherits` 的值 **= 命名空间.类名**
- 前半部分（如 `JHSoft.Web.WorkFlat`）对应 `bin/` 中的 dll 文件名
- 后半部分（`DBModules`）即类名 / 后端控制类
- `CodeBehind` 仅在源码工程中有效；若仅有 dll，必须**反编译** `JHSoft.Web.WorkFlat.dll` 中的 `DBModules` 类

### 0.3 看 web.config

- 检查 `<system.web>` / `<authentication>` / `<authorization>` 节，了解鉴权方式（用于路由的"无鉴权"标注）
- 检查 `<httpHandlers>` / `<handlers>` 节，识别自定义 `IHttpHandler` 注册的 URL → 类映射

### 0.4 看 bin 目录与命名空间

- 闭源项目：先反编译 `bin/*.dll`，全局搜索 `namespace *.Controllers` 定位控制层
- 注意：`System.*` 命名空间下属于 .NET 自带类库，必须排除
- 全局搜索 `RegisterRoutes` / `MapRoute` / `MapControllerRoute` / `MapPageRoute` / `Application_Start`

### 0.5 框架与非框架的快速判定

| 形态 | 判定 |
| --- | --- |
| `bin/*.dll` + `Views/*.cshtml` + 路径形如 `/Home/Index` | **MVC 框架** |
| 散落的 `.aspx` 物理文件，URL 直接对应文件路径 | **Web Forms（非框架）** |

---

## 1. ASP.NET MVC 5 及以下版本路由提取

### 1.1 路由配置位置

- 主文件：`App_Start/RouteConfig.cs` 中的 `RegisterRoutes(RouteCollection routes)` 方法
- 注册入口：`Global.asax.cs` 的 `Application_Start` 中调用 `RouteConfig.RegisterRoutes(RouteTable.Routes)`

```csharp
// App_Start/RouteConfig.cs
public class RouteConfig
{
    public static void RegisterRoutes(RouteCollection routes)
    {
        routes.IgnoreRoute("{resource}.axd/{*pathInfo}");
        routes.MapRoute(
            name: "Default",
            url: "{controller}/{action}/{id}",
            defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional }
        );
    }
}
```

### 1.2 提取步骤

1. 解析 `RouteConfig.cs` 中所有 `MapRoute(...)` 调用，记录每条路由模板（含 `name` / `url` / `defaults` / `constraints`）
2. 从 `defaults` 推导默认 controller / action
3. 全局搜索继承自 `Controller` / `ApiController` 的类（命名空间常含 `.Controllers`）
4. 对每个 Controller，列举所有 `public` 方法作为 Action（排除 `[NonAction]`）
5. 检查每个 Action 上的属性：`[HttpGet]`/`[HttpPost]`/`[Route(...)]`/`[ActionName(...)]`/`[Authorize]`/`[AllowAnonymous]`
6. 通过约定路由模板（`{controller}/{action}/{id}`）+ 属性路由叠加生成最终 URL
7. 对每个 Action 的形参，按以下规则归类参数来源：
   - `[FromBody]` 标注或为复杂模型 → Body
   - `[FromUri]`/`[FromQuery]` 或基本类型 → Query / RouteValue
   - 出现在 URL 模板的占位符 → Path
   - `HttpContext.Request.Form[...]` / `Request["..."]` → Form / Mixed
8. 闭源项目无 `RouteConfig.cs` 时，**先反编译 dll**，再按上述规则执行

### 1.3 关键搜索词

```
MapRoute / RegisterRoutes / RouteTable.Routes / IgnoreRoute
: Controller / : ApiController
[Route( / [HttpGet( / [HttpPost( / [HttpPut( / [HttpDelete(
[RoutePrefix(
```

---

## 2. ASP.NET Core (.NET 5/6/7+) 路由提取

### 2.1 路由配置位置

- `Program.cs`（.NET 6+）或 `Startup.cs::Configure`（.NET Core 3.1）
- 关键中间件：`app.UseRouting()` + `app.UseEndpoints(...)` 或最小 API `app.MapGet/MapPost/...`

```csharp
// Program.cs (.NET 6+) 约定路由
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.UseRouting();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
app.Run();
```

```csharp
// 属性路由（推荐写法）
[Route("api/[controller]")]
public class ProductsController : Controller
{
    [HttpGet("{id}")]                 // GET  /api/products/{id}
    public IActionResult GetProduct(int id) { ... }

    [HttpPost]                         // POST /api/products
    public IActionResult Create([FromBody] Product p) { ... }
}
```

### 2.2 提取步骤

1. 解析 `Program.cs` / `Startup.cs`：
   - `MapControllerRoute` / `MapDefaultControllerRoute` / `MapAreaControllerRoute` → 记录约定路由模板
   - `MapControllers()` → 表示纯属性路由
   - `app.MapGet/MapPost/MapPut/MapDelete/MapMethods` → 记录最小 API 端点
2. 全局搜索继承 `ControllerBase` / `Controller` 的类
3. 提取类级 `[Route("...")]`、`[ApiController]`、`[Area("...")]`
4. 提取方法级 `[HttpGet("...")]` / `[HttpPost("...")]` / `[Route("...")]` / `[AcceptVerbs(...)]`
5. URL = 类级 Route + 方法级 Route，并展开 token：
   - `[controller]` → 类名去除 `Controller` 后缀
   - `[action]` → 方法名
   - `[area]` → 区域名
6. 参数分析：
   - `[FromBody]` / `[FromForm]` / `[FromQuery]` / `[FromRoute]` / `[FromHeader]` / `[FromServices]`
   - 未标注 + 简单类型 → 默认 Query / Route（依模板）
   - 未标注 + 复杂类型 → 默认 Body（API Controller）或 Form（MVC）
7. `[Authorize]` / `[AllowAnonymous]` 必须记录在每个路由上，供下游鉴权 skill 使用

### 2.3 关键搜索词

```
WebApplication.CreateBuilder
UseRouting / UseEndpoints / MapControllers / MapControllerRoute
MapGet( / MapPost( / MapPut( / MapDelete( / MapMethods(
[ApiController] / [Route( / [HttpGet( / [HttpPost(
[Authorize / [AllowAnonymous
```

---

## 3. ASP.NET Web Forms 路由提取

### 3.1 两种路由形态

#### 3.1.1 物理文件路径（默认）

每个 `.aspx` 文件本身就是一个路由，URL 直接为相对路径：
```
/Default.aspx
/admin/UserList.aspx
/Contact.aspx?id=123
```

#### 3.1.2 友好 URL（手动注册）

在 `Global.asax.cs::Application_Start` 中通过 `MapPageRoute` 注册：
```csharp
protected void Application_Start(object sender, EventArgs e)
{
    RouteTable.Routes.MapPageRoute(
        "ProductsRoute",            // 路由名称
        "products/{category}",       // URL 模式
        "~/Products.aspx"           // 实际页面
    );
}
```

### 3.2 提取步骤

1. **枚举所有 `.aspx` / `.ashx` / `.asmx`** 物理文件，记录其相对路径作为基础 URL
2. 解析每个 `.aspx` 顶部 `<%@ Page %>` 指令：
   - `Inherits="X.Y.Z.ClassName"` → 后端类全名
   - `CodeBehind="X.aspx.cs"` → 源码文件（无源码时反编译 dll）
3. **反编译/查看后端类**，提取：
   - `Page_Load(object sender, EventArgs e)`：页面加载逻辑
   - 控件事件方法：`btnSubmit_Click` / `gridView_RowCommand` / `LinkButton1_Click` 等
   - `IHttpHandler.ProcessRequest`（针对 .ashx）
4. 参数分析：
   - `Request.QueryString["x"]` → Query 参数 `x`
   - `Request.Form["y"]` → Form 参数 `y`
   - `Request["z"]` → Mixed（QueryString ∪ Form ∪ Cookies ∪ ServerVariables）
   - 控件 `txtUser.Text` / `ddlRole.SelectedValue` → 间接 Form（控件 `name=ctl00$...` 形式）
   - `Request.Cookies` / `Request.Headers` → Cookie / Header
5. 解析 `Global.asax.cs` 中所有 `MapPageRoute(...)`，把 URL 模式与目标 `.aspx` 关联
6. 解析 `web.config` 的 `<httpHandlers>`/`<handlers>`，记录自定义 Handler URL
7. **对 `.asmx` Web Service**，列出所有标注 `[WebMethod]` 的方法，URL 形如 `/Service.asmx/MethodName`，参数即方法形参

### 3.3 关键搜索词

```
<%@ Page  /  <%@ Control  /  <%@ WebService  /  <%@ WebHandler
Inherits="
MapPageRoute
Page_Load / OnLoad
Request.QueryString / Request.Form / Request.Cookies / Request.Headers
[WebMethod]
IHttpHandler / ProcessRequest
```

---

## 4. 闭源 dll 反编译流程（无 .cs 源码时）

完整流程详见 `references/DECOMPILE_STRATEGY.md`，要点：

1. **混淆识别（前置）**：先抽样查看 `bin/` 中目标 dll，若出现 `\u0001` / `\u0002` 之类成员名、单字符命名空间、或 `strings <dll> | grep -i confuser` 命中混淆器水印，说明已被混淆 → 跳到第 2 步；否则跳到第 3 步
2. **去混淆（仅当混淆时）**：用 `de4dot` 整批清洗 `bin/` → `bin_clean/`，详见 `references/DECOMPILE_STRATEGY.md` §8~§9（含批量 Python 脚本），后续 ilspycmd 用 `bin_clean/`
3. 反编译工具：**`ilspycmd`**（统一使用，跨平台 CLI；安装：`dotnet tool install -g ilspycmd`）
4. 批量反编译：把 `bin/`（或脱壳后的 `bin_clean/`）下**全部**项目 dll 用 `ilspycmd <dll> -p --referencepath bin -o {project}_audit/decompiled/<name>/` 落盘为 C# 项目
5. 定位入口：
   - 全局搜索 `RegisterRoutes` / `Application_Start` / `MapPageRoute` / `MapControllerRoute`
   - 全局搜索 `namespace *.Controllers`，过滤 `System.*`
6. 仅有 `Global.asax` 而无路由配置类时，先反编译 `Global.asax` 对应的 dll，确认其调用的 `RouteConfig` 类
7. 闭源 Web Forms：用 `.aspx` 中的 `Inherits` 值反查 dll → 类 → 方法
8. 反编译产物以 `.cs` 形式落盘到 `decompiled/` 目录，作为后续静态分析输入

---

## 5. 输出规范

输出统一保存到 `{project}_audit/route_mapper/` 目录。

### 5.1 目录结构

```
{project}_audit/route_mapper/
├── README.md                  # 主索引：项目类型、路由总数、模块分布
├── routes_index.md            # 全部路由清单（按模块分组的精简表格）
├── modules/
│   ├── {ModuleA}.md          # 每模块一份详细路由
│   ├── {ModuleB}.md
│   └── ...
└── webforms_pages.md          # 仅 Web Forms 项目：物理 .aspx 列表
```

### 5.2 单条路由的最小信息字段

每条路由必须给出：

| 字段 | 说明 |
| --- | --- |
| 序号 | 全局唯一编号 |
| 名称 | 推断的功能名（如 "用户登录"） |
| 入口位置 | `Namespace.Class.Method` + 文件相对路径:行号（或反编译产物路径） |
| HTTP 方法 | GET / POST / PUT / DELETE / ANY |
| URL 路径 | 已展开 token 的最终路径 |
| 路由形态 | Convention / Attribute / MapPageRoute / Physical / MinimalAPI |
| 参数结构 | Query / Form / Body / Path / Header / Cookie / RouteValue 列表 |
| 鉴权标注 | `[Authorize]` / `[AllowAnonymous]` / 无 / 来自 `web.config` 的规则 |
| 备注 | 是否反编译得到、特殊属性等 |

### 5.3 接口区块紧凑格式

每条路由按下面格式输出（参数使用**紧凑单行**，省略 token）：

```
=== [N] 用户登录 ===
位置: WebApplication1.Controllers.AccountController.Login
       (Controllers/AccountController.cs:42 或 decompiled/WebApplication1/Controllers/AccountController.cs:42)
HTTP 方法: POST
URL 路径: /Account/Login
路由形态: Convention（{controller}/{action}/{id}）
鉴权: [AllowAnonymous]
参数: Body(form-urlencoded): userName:string, password:string, returnUrl:string
```

参数书写规范（仅在确有参数时写，无参数填 `无`）：
- 多类型时按 `Path: ... | Query: ... | Body(...): ... | Form: ... | Header: ... | Cookie: ... | RouteValue: ...` 顺序拼接
- `Body(...)` 括号内写 Content-Type 简称：`json` / `form-urlencoded` / `multipart` / `xml`
- **仅在以下情形单独列出 Header / Content-Type 行**：
  - `multipart/form-data`（文件上传）
  - SOAP / xml
  - 自定义鉴权头（如 `X-Auth-Token`）且非标准 `Authorization`
- 不输出 HTTP 请求模板

### 5.4 Web Forms 物理页面列表

```
=== Physical .aspx Pages ===
| 序号 | 页面 | URL | 后端类 | dll | 鉴权 |
|------|------|------|--------|-----|------|
| 1 | Default.aspx | /Default.aspx | WebApplication1.Default | WebApplication1.dll | 无 |
| 2 | admin/UserList.aspx | /admin/UserList.aspx | WebApplication1.Admin.UserList | WebApplication1.dll | <authorization> 拒绝匿名 |
```

详细输出模板见 `references/OUTPUT_TEMPLATE.md`。

---

## 6. 执行 Checklist

完成后逐项核对：

- [ ] 已识别项目类型（MVC5 / Core / WebForms / WebAPI / 最小 API / 混合）
- [ ] 已记录所有 `RegisterRoutes` / `MapRoute` / `MapControllerRoute` / `MapPageRoute` / `MapGet*` 调用
- [ ] 已枚举所有 Controller（含反编译产物中的）
- [ ] 已枚举所有 Action（含 `[Route]` / `[HttpGet]` / `[HttpPost]` / `[NonAction]`）
- [ ] Web Forms 已枚举所有 `.aspx` / `.ashx` / `.asmx` 物理文件
- [ ] Web Forms 已解析每个 `.aspx` 的 `Inherits` 与对应 dll
- [ ] 闭源 dll 已全部反编译并入库
- [ ] 每条路由都已分析参数来源（Query / Form / Body / Path / Header / Cookie）
- [ ] 每条路由都已记录 `[Authorize]` / `[AllowAnonymous]` 标注
- [ ] 输出文件按 `{project}_audit/route_mapper/` 目录保存
- [ ] 路由总数与清单一致，未省略

---

## 7. 引用文档

- `references/ASPNET_MVC5.md`：MVC 5 / Web API 2 详细规则
- `references/ASPNET_CORE.md`：ASP.NET Core / 最小 API 详细规则
- `references/ASPNET_WEBFORMS.md`：Web Forms 详细规则
- `references/DECOMPILE_STRATEGY.md`：dll 反编译策略
- `references/OUTPUT_TEMPLATE.md`：标准输出模板
