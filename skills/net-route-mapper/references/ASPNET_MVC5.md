# ASP.NET MVC 5 / Web API 2 路由提取细则

## 1. 路由载体

| 载体 | 文件 | 关键 API |
| --- | --- | --- |
| MVC 5 约定路由 | `App_Start/RouteConfig.cs` | `RouteCollection.MapRoute(...)` / `IgnoreRoute(...)` |
| Web API 2 约定路由 | `App_Start/WebApiConfig.cs` | `HttpRouteCollection.MapHttpRoute(...)` |
| 注册入口 | `Global.asax.cs` | `Application_Start()` 中调用 `RouteConfig.RegisterRoutes(RouteTable.Routes)` |
| 属性路由 | Controller 类 / Action 方法 | `[RoutePrefix]` / `[Route]` / `[HttpGet]` / `[HttpPost]` 等 |

## 2. 约定路由模板示例

```csharp
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

`Global.asax.cs`：

```csharp
protected void Application_Start()
{
    AreaRegistration.RegisterAllAreas();
    RouteConfig.RegisterRoutes(RouteTable.Routes);
    GlobalConfiguration.Configure(WebApiConfig.Register);
}
```

## 3. 提取规则

1. **每条 `MapRoute` / `MapHttpRoute`** 都要单独记录：
   - `name` / `url` / `defaults` / `constraints`
2. 使用模板展开 URL：
   - `{controller}` → 类名去掉 `Controller` 后缀
   - `{action}` → 方法名
   - `{id}` → Path 参数
3. **属性路由优先级高于约定路由**。同一 Action 同时存在两种时按属性路由输出最终 URL。
4. `[RoutePrefix("api/users")]` + `[Route("{id:int}")]` → URL = `/api/users/{id:int}`
5. `[Route("~/...")]`（前缀 `~/`）会忽略 `RoutePrefix`，直接使用绝对路径。
6. **Areas**（`/Areas/Admin/AdminAreaRegistration.cs`）：URL 前缀为 `Admin/`。
7. `[NonAction]`、私有方法、构造函数、`Dispose` 等不算 Action。

## 4. 参数解析

| 参数标注 | 来源 |
| --- | --- |
| `[FromBody]` | Body（JSON / XML，根据 Content-Type 协商） |
| `[FromUri]` | Query + Route |
| 简单类型默认 | Query（API）/ Route + Query（MVC） |
| 复杂类型默认 | Body（API）/ Form（MVC） |
| 出现在 URL 模板 | Path / Route |
| `HttpContext.Request.Form[...]` | Form |
| `HttpContext.Request.QueryString[...]` | Query |
| `HttpContext.Request["..."]` | Mixed |

## 5. 鉴权属性

- 类级：`[Authorize]` / `[Authorize(Roles="Admin")]`
- 方法级：`[Authorize]` / `[AllowAnonymous]`
- 全局过滤器：`FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters)` 中 `filters.Add(new AuthorizeAttribute())`

输出每条路由时必须明确标注鉴权状态：`[Authorize]` / `[AllowAnonymous]` / `Global` / `None`。

## 6. 关键搜索词

```
RegisterRoutes
MapRoute(
MapHttpRoute(
IgnoreRoute(
RouteTable.Routes
GlobalConfiguration.Configure
: Controller
: ApiController
[RoutePrefix(
[Route(
[HttpGet( / [HttpPost( / [HttpPut( / [HttpDelete( / [HttpPatch(
[ActionName(
[NonAction]
[Authorize / [AllowAnonymous
AreaRegistration
```

## 7. 闭源场景

- `App_Start/RouteConfig.cs` 通常被编译进主 dll，反编译后定位 `RegisterRoutes` 静态方法即可。
- 全局搜索 `RouteCollection` / `HttpRouteCollection` 类型的扩展方法调用，可补全遗漏。
