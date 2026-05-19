# .NET 鉴权模式识别参考

按项目类型选择相关章节。只把本文件作为搜索与判据清单，不要把它完整复制进报告。

## ASP.NET Core / Minimal API

关键入口：
- `Program.cs`、`Startup.cs`
- `builder.Services.AddAuthentication`
- `builder.Services.AddAuthorization`
- `app.UseAuthentication()`
- `app.UseAuthorization()`
- `MapControllers`、`MapControllerRoute`、`MapGet`、`MapPost`、`MapGroup`
- `RequireAuthorization()`、`AllowAnonymous()`
- `FallbackPolicy`、`DefaultPolicy`
- `AddJwtBearer`、`AddCookie`
- `[Authorize]`、`[AllowAnonymous]`、`[Authorize(Roles=...)]`、`[Authorize(Policy=...)]`

必须判断：
- `UseAuthentication()` 是否在 `UseAuthorization()` 之前。
- 是否存在全局 `FallbackPolicy`，避免误报缺少 `[Authorize]` 的接口。
- Minimal API 是否通过 `MapGroup(...).RequireAuthorization()` 或单端点 `RequireAuthorization()` 保护。
- `[AllowAnonymous]` 或 `AllowAnonymous()` 是否覆盖了类级、组级或全局授权。
- JWT 校验是否只认证身份，资源归属是否另有服务层校验。

搜索词：

```text
AddAuthentication
AddAuthorization
AddJwtBearer
AddCookie
UseAuthentication
UseAuthorization
FallbackPolicy
DefaultPolicy
RequireAuthorization
AllowAnonymous
[Authorize
[AllowAnonymous
IAuthorizationHandler
AuthorizationHandler
IAsyncAuthorizationFilter
AuthorizeFilter
```

常见漏点：
- MapGroup 级别授权被忽略。
- FallbackPolicy 导致无 `[Authorize]` 的端点仍受保护。
- 自定义 `IAuthorizationHandler` 才是真正的资源级授权入口。
- `appsettings.json` 中 JWT issuer/audience/key 配置影响认证行为，但不是权限充分性的证据。

## ASP.NET MVC 5 / Web API 2

关键入口：
- `App_Start/FilterConfig.cs`
- `App_Start/WebApiConfig.cs`
- `App_Start/RouteConfig.cs`
- `Global.asax` / `Application_Start`
- `GlobalFilters.Filters.Add(new AuthorizeAttribute())`
- `config.Filters.Add(...)`
- `AuthorizeAttribute`、`AllowAnonymousAttribute`
- 自定义 `ActionFilterAttribute`、`AuthorizationFilterAttribute`、`IAuthenticationFilter`
- Controller 基类的 `OnActionExecuting`

必须判断：
- 是否存在全局 `AuthorizeAttribute`。
- Action 缺少 `[Authorize]` 时，类级、基类、全局过滤器是否已保护。
- `[AllowAnonymous]` 是否显式放行敏感 Action。
- Web API 过滤器和 MVC 过滤器是否分别配置，不能混为一谈。

搜索词：

```text
GlobalFilters.Filters.Add
new AuthorizeAttribute
config.Filters.Add
RegisterGlobalFilters
FilterConfig.RegisterGlobalFilters
[Authorize
[AllowAnonymous
AuthorizeAttribute
AllowAnonymousAttribute
ActionFilterAttribute
AuthorizationFilterAttribute
IAuthenticationFilter
OnActionExecuting
User.Identity.IsAuthenticated
Request.IsAuthenticated
```

常见漏点：
- MVC 和 Web API 有各自过滤器管线。
- Controller 基类可能在 `OnActionExecuting` 中统一校验登录或权限。
- 自定义 Attribute 名称不一定包含 Auth，需结合返回 401/403、Redirect、Session/User 判断。

## ASP.NET Web Forms

关键入口：
- `web.config <authentication>`、`<authorization>`、`<location path=...>`
- `.aspx` 顶部 `<%@ Page Inherits="..." %>`
- Page 基类、母版页、用户控件
- `OnInit`、`Page_Load`、`OnLoad`
- `FormsAuthentication`、Windows 鉴权、Session 校验

必须判断：
- `web.config` 是否全局拒绝匿名：`<deny users="?" />`。
- `location path` 是否对目录或页面单独放行/限制。
- `.aspx` 的 `Inherits` 是否指向 dll 中的后端类，闭源时必须在反编译产物中追踪。
- Page 基类或生命周期方法是否统一执行鉴权。

搜索词：

```text
<authentication
<authorization
<location
deny users="?"
allow users="*"
FormsAuthentication
Session[
Page_Load
OnInit
OnLoad
Request.IsAuthenticated
Context.User
User.Identity
Inherits="
```

常见漏点：
- 只看 `.aspx` 文件路径，没追 `Inherits` 对应类和基类。
- MVC 项目中也可能存在 `web.config` 鉴权或 Web Forms 混用页面。
- `OnInit` 比 `Page_Load` 更早执行，常被用作统一鉴权入口。

## Handler / ASMX / 其他入口

关键入口：
- `.ashx` / `IHttpHandler.ProcessRequest`
- `.asmx` / `[WebMethod]`
- 自定义 `HttpModule`
- `IHttpModule.Init`
- `web.config <handlers>`、`<httpHandlers>`、`<modules>`

搜索词：

```text
IHttpHandler
ProcessRequest
IHttpModule
Init(HttpApplication
[WebMethod]
ScriptService
<handlers
<httpHandlers
<modules
```

常见漏点：
- Handler 不经过 MVC Controller 过滤器，必须单独确认鉴权。
- ASMX 方法可能直接暴露给脚本调用。
- HttpModule 可能是全局认证入口，也可能只做日志或兼容处理，不能凭名称判断。

## 证据要求

每条鉴权结论至少包含一种证据：
- Attribute 证据：类/方法上的 `[Authorize]`、`[AllowAnonymous]`、自定义权限 Attribute。
- 全局证据：FallbackPolicy、GlobalFilters、web.config authorization、MapGroup RequireAuthorization。
- 代码证据：Session/User/Claims/Role/Policy/权限服务调用。
- 继承证据：Controller/Page 基类或生命周期方法。
- 反编译证据：manifest 状态、反编译路径、类/方法位置。

没有证据时只能标为“不确定”，不得直接判定安全或漏洞。
