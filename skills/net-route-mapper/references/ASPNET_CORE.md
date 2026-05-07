# ASP.NET Core / 最小 API 路由提取细则

## 1. 路由载体

| 载体 | 文件 | 关键 API |
| --- | --- | --- |
| 顶级配置（.NET 6+） | `Program.cs` | `WebApplication.CreateBuilder` / `app.MapControllers()` / `app.MapControllerRoute(...)` |
| Startup 配置（.NET Core 3.1） | `Startup.cs::Configure` | `app.UseRouting()` + `app.UseEndpoints(...)` |
| 属性路由 | Controller / Action | `[Route]` / `[ApiController]` / `[HttpGet]` / `[HttpPost]` |
| 最小 API | `Program.cs` | `app.MapGet/MapPost/MapPut/MapDelete/MapMethods` |
| Razor Pages | `Pages/*.cshtml` + `*.cshtml.cs` | `[BindProperty]` / `OnGet/OnPost` |
| SignalR Hub | Hub 类 | `app.MapHub<TChat>("/chathub")` |
| gRPC | proto + `Service` 类 | `app.MapGrpcService<...>()` |

## 2. 约定路由示例

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.MapAreaControllerRoute(
    name: "areas",
    areaName: "Admin",
    pattern: "Admin/{controller=Home}/{action=Index}/{id?}");

app.MapControllers();          // 启用属性路由
app.MapRazorPages();
app.MapHub<ChatHub>("/chat");
app.MapGet("/health", () => "OK");

app.Run();
```

## 3. 属性路由示例

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]                          // GET  /api/products
    public IActionResult List() { ... }

    [HttpGet("{id:int}")]              // GET  /api/products/{id}
    public IActionResult Get(int id) { ... }

    [HttpPost]                         // POST /api/products
    public IActionResult Create([FromBody] ProductDto dto) { ... }

    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    public IActionResult Delete(int id) { ... }
}
```

## 4. 提取规则

1. 类级路由前缀：`[Route("api/[controller]")]` / `[RoutePrefix(...)]`（兼容性）
2. token 展开：
   - `[controller]` → 类名去除 `Controller` 后缀
   - `[action]` → 方法名
   - `[area]` → 区域名
3. 方法上的所有 HTTP 谓词：`[HttpGet]` / `[HttpPost]` / `[HttpPut]` / `[HttpDelete]` / `[HttpPatch]` / `[AcceptVerbs("GET","POST")]` / `[Route]`（多谓词）
4. 同一 Action 出现多个 `[Route]` / `[HttpXxx("...")]` → 输出**多条**路由
5. 路由约束：`{id:int}` / `{slug:regex(...)}` 等需要保留在 URL 中
6. **最小 API**：每个 `app.MapGet("/path", handler)` 单独记录一条路由，handler 形参分析按 `[FromXxx]` 规则
7. **Razor Pages**：URL = 物理路径（去除 `/Pages` 前缀和 `.cshtml` 后缀），方法形态为 `OnGet`/`OnPost`/`OnGetAsync` 等，BindProperty 字段视为 Form/Query
8. **SignalR Hub**：URL 为 `MapHub<>` 的路径，方法即 Hub 公有方法

## 5. 参数标注

| 标注 | 含义 |
| --- | --- |
| `[FromBody]` | Body（JSON 默认） |
| `[FromForm]` | Form（multipart/x-www-form-urlencoded） |
| `[FromQuery]` | Query string |
| `[FromRoute]` | Route 模板占位符 |
| `[FromHeader]` | 请求头 |
| `[FromServices]` | DI，非用户输入，**不算请求参数** |
| `[BindRequired]` / `[BindNever]` | 模型绑定控制 |
| 无标注 + `[ApiController]` + 简单类型 | Query 或 Route |
| 无标注 + `[ApiController]` + 复杂类型 | Body |
| 无标注 + 普通 MVC + 复杂类型 | Form / Query 混合 |

## 6. 鉴权属性

- `[Authorize]` / `[Authorize(Policy="...")]` / `[Authorize(Roles="...")]`
- `[AllowAnonymous]`
- `app.UseAuthorization()` 之后 `MapXxx().RequireAuthorization()` 链式调用
- 最小 API：`app.MapGet("/x", ...).RequireAuthorization()`

## 7. 关键搜索词

```
WebApplication.CreateBuilder
UseRouting / UseEndpoints
MapControllers( / MapControllerRoute( / MapAreaControllerRoute( / MapDefaultControllerRoute(
MapRazorPages( / MapHub< / MapGrpcService<
MapGet( / MapPost( / MapPut( / MapDelete( / MapPatch( / MapMethods(
[ApiController]
[Route( / [HttpGet( / [HttpPost( / [HttpPut( / [HttpDelete( / [HttpPatch(
[FromBody / [FromQuery / [FromRoute / [FromForm / [FromHeader
[Authorize / [AllowAnonymous
RequireAuthorization
```

## 8. 闭源场景

- 顶级语句（top-level statements）会被编译为 `Program` 类的 `<Main>$` 方法，反编译后阅读该方法。
- 中间件 / `MapXxx` 调用通过扩展方法存在，反编译时关注调用链尾部的字符串字面量参数（即 URL 模板）。
