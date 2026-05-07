# 输出模板

所有输出统一到 `{project}_audit/route_mapper/`。

## 1. 目录结构

```
{project}_audit/route_mapper/
├── README.md                  # 主索引
├── routes_index.md            # 路由清单（精简表格）
├── modules/
│   ├── {ModuleA}.md
│   ├── {ModuleB}.md
│   └── ...
└── webforms_pages.md          # 仅 Web Forms 项目
```

## 2. README.md 模板

```markdown
# {项目名} - 路由映射主索引

## 项目识别
- 项目类型: ASP.NET Core 6 / ASP.NET MVC 5 / Web Forms / 混合
- 入口程序集: WebApplication1.dll
- 路由载体: Program.cs / RouteConfig.cs / Global.asax
- 反编译产物: decompiled/

## 鉴权概览
- 全局过滤器: 无 / [Authorize] (FilterConfig)
- web.config <authorization>: 见各模块

## 路由统计
- 总路由数: {N}
- 模块数: {M}
- HTTP 方法分布:
  - GET:    {x}
  - POST:   {y}
  - PUT:    {z}
  - DELETE: {w}
  - 其他:   {v}

## 模块列表
| 模块 | 路由数 | 文件 |
|------|--------|------|
| Account | 12 | modules/Account.md |
| Admin | 25 | modules/Admin.md |
| Api/Products | 8 | modules/Api_Products.md |

## Web Forms 物理页面
见 webforms_pages.md（共 {K} 个 .aspx / .ashx / .asmx）
```

## 3. routes_index.md 模板（精简表格）

```markdown
# 路由清单（共 {N} 条）

| # | HTTP | URL | 入口 | 鉴权 | 模块 |
|---|------|-----|------|------|------|
| 1 | POST | /Account/Login | AccountController.Login | [AllowAnonymous] | Account |
| 2 | GET  | /Account/Profile | AccountController.Profile | [Authorize] | Account |
| 3 | GET  | /api/products/{id} | ProductsController.Get | [Authorize] | Api/Products |
| 4 | -    | /Default.aspx | WebApplication1.Default | <authorization> 默认 | WebForms |
```

## 4. modules/{Module}.md 模板（详细，紧凑格式）

> **硬约束：**
> 1. 必须列出该模块下所有路由，不得省略任何路由
> 2. 每个路由必须有完整位置信息和参数行
> 3. 参数使用**紧凑单行**格式：`Path: ... | Query: ... | Body(...): ... | Form: ... | Header: ... | Cookie: ...`
> 4. **不输出 HTTP / Burp 请求模板**
> 5. Content-Type / Header 仅在文件上传 / SOAP / 自定义鉴权头时单独列出

```markdown
# 模块: {模块名}（共 {n} 条路由）

## 鉴权概览
- 类级: [Authorize] / 无
- 全局: 见 README.md

---

=== [1] 用户登录 ===
位置: WebApplication1.Controllers.AccountController.Login
       (Controllers/AccountController.cs:42 或 decompiled/WebApplication1/Controllers/AccountController.cs:42)
HTTP 方法: POST
URL 路径: /Account/Login
路由形态: Convention（{controller}/{action}/{id}）
鉴权: [AllowAnonymous]
参数: Body(form-urlencoded): userName:string, password:string, returnUrl:string

=== [2] 获取产品 ===
位置: WebApplication1.Controllers.Api.ProductsController.Get
       (Controllers/Api/ProductsController.cs:35)
HTTP 方法: GET
URL 路径: /api/products/{id:int}
路由形态: Attribute（[Route("api/[controller]")] + [HttpGet("{id:int}")]）
鉴权: [Authorize]
参数: Path: id:int

=== [3] 文件上传 ===
位置: WebApplication1.Controllers.UploadController.Upload
       (Controllers/UploadController.cs:18)
HTTP 方法: POST
URL 路径: /Upload/Upload
路由形态: Convention
鉴权: [Authorize]
Content-Type: multipart/form-data
参数: Form: file:IFormFile, category:string

---

## 模块统计

| 统计项 | 数量 |
|--------|------|
| 总路由数 | {n} |
| MVC 路由 | {a} |
| Web API 路由 | {b} |
| 最小 API 路由 | {c} |
| Web Forms 页面 | {d} |
```

## 5. webforms_pages.md 模板（仅 Web Forms 项目）

```markdown
# Web Forms 物理页面（共 {K} 个）

## 鉴权来源
- 来自 web.config <authorization> 节，按 <location path> 规则匹配

## 页面清单

| # | 类型 | URL | 后端类 | 程序集 | 鉴权 |
|---|------|-----|--------|--------|------|
| 1 | .aspx | /Default.aspx | WebApplication1.Default | WebApplication1.dll | 默认 allow * |
| 2 | .aspx | /admin/UserList.aspx | WebApplication1.Admin.UserList | WebApplication1.dll | <location path="admin"> deny ? |
| 3 | .ashx | /Upload.ashx | WebApplication1.Handlers.Upload | WebApplication1.dll | 无 |
| 4 | .asmx | /UserSvc.asmx | WebApplication1.Services.UserSvc | WebApplication1.dll | 无 |

## 友好 URL 映射（MapPageRoute）

| 路由名 | URL 模式 | 实际页面 |
|--------|----------|----------|
| ProductsRoute | products/{category} | ~/Products.aspx |

## 详细页面（每个 .aspx 一段，紧凑格式）

=== [1] Default.aspx ===
后端类: WebApplication1.Default (extends System.Web.UI.Page)
程序集: WebApplication1.dll
鉴权: 默认 allow *
入口方法:
  - Page_Load(object sender, EventArgs e)
  - btnSubmit_Click(object sender, EventArgs e)
参数: Form: ctl00$txtUser:string, ctl00$txtPwd:string | Query: redirect:string

=== [2] admin/UserList.aspx ===
后端类: WebApplication1.Admin.UserList
程序集: WebApplication1.dll
鉴权: <location path="admin"> deny ?
入口方法:
  - Page_Load
  - gridView_RowCommand
参数: Query: page:int, size:int

=== [3] Upload.ashx ===
后端类: WebApplication1.Handlers.Upload (implements IHttpHandler)
程序集: WebApplication1.dll
鉴权: 无
入口方法: ProcessRequest(HttpContext)
Content-Type: multipart/form-data
参数: Form: file:HttpPostedFile, name:string

=== [4] UserSvc.asmx / Login ===
后端类: WebApplication1.Services.UserSvc.Login (extends WebService)
程序集: WebApplication1.dll
鉴权: 无
URL 路径: /UserSvc.asmx/Login
参数: Body(form-urlencoded 或 SOAP): user:string, pass:string
```

## 6. 完整性检查输出

每次执行结束追加一段：

```markdown
## 完整性检查

- [x] 项目类型已识别
- [x] 所有 RegisterRoutes / MapRoute / MapPageRoute / MapControllerRoute / MapGet 已覆盖
- [x] 所有 Controller / Action 已枚举
- [x] 所有 .aspx / .ashx / .asmx 已枚举
- [x] 所有路由含参数行与鉴权标注
- [x] 路由总数 = routes_index.md 行数 = 各模块路由数之和
- [x] 未输出 HTTP / Burp 请求模板（按规范禁止）
```
