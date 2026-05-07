# ASP.NET Web Forms 路由提取细则

## 1. 三种路由形态

| 形态 | 说明 | URL 形如 |
| --- | --- | --- |
| 物理文件 | 每个 `.aspx`/`.ashx`/`.asmx` 自身就是一个端点 | `/admin/UserList.aspx` |
| 友好 URL | `Global.asax::Application_Start` 中 `MapPageRoute` 注册 | `/products/{category}` |
| HttpHandler | `web.config` 的 `<httpHandlers>`/`<handlers>` 中注册 | `/Upload.do` |

## 2. .aspx 顶部声明解析

```aspx
<%@ Page Language="C#" AutoEventWireup="true"
         CodeBehind="Contact.aspx.cs"
         Inherits="WebApplication1.Contact" %>
```

| 字段 | 含义 |
| --- | --- |
| `Language` | 后端语言（C# / VB） |
| `AutoEventWireup` | 是否自动绑定事件（true 时 `Page_Load` 自动挂接） |
| `CodeBehind` | 后端源码文件名（仅源码工程有效） |
| `Inherits` | 后端类全名 = `命名空间.类名` |

提取规则：
- `Inherits` 的命名空间前缀 → 对应 `bin/` 中的 dll 名（如 `WebApplication1.Contact` → `bin/WebApplication1.dll`）
- 若无源码，必须**反编译该 dll** 中的 `Contact` 类
- 该类继承 `System.Web.UI.Page`（普通页）或 `IHttpHandler`（.ashx）或 `WebService`（.asmx）

## 3. Code-Behind 后端入口

| 入口方法 | 触发时机 |
| --- | --- |
| `Page_Load` | 页面加载（每次 GET / POST 都会进） |
| `Page_Init` / `Page_PreRender` | 生命周期其他阶段 |
| `控件_Click` 等 | 控件回调（POST 中 `__EVENTTARGET` 指向该控件） |
| `OnInit` 重写 | 初始化阶段重写 |
| `IHttpHandler.ProcessRequest` | .ashx 唯一入口 |
| 标注 `[WebMethod]` 的方法 | .asmx Web Service 方法 |

## 4. 友好 URL 注册

```csharp
protected void Application_Start(object sender, EventArgs e)
{
    RouteTable.Routes.MapPageRoute(
        "ProductsRoute",
        "products/{category}",
        "~/Products.aspx"
    );
}
```

提取要点：
- 参数 1：路由名
- 参数 2：URL 模式（含 `{...}` 占位符）
- 参数 3：物理 .aspx 路径
- 输出时 URL 用模式 + 占位符表示，并附"实际页面"指向物理文件

## 5. 参数提取

Web Forms 没有形参声明，必须扫描后端类中所有：

| 调用 | 参数来源 |
| --- | --- |
| `Request.QueryString["x"]` | Query |
| `Request.Form["y"]` | Form |
| `Request["z"]` | Mixed（QueryString ∪ Form ∪ Cookies ∪ ServerVariables） |
| `Request.Cookies["c"]` | Cookie |
| `Request.Headers["h"]` | Header |
| `Page.RouteData.Values["v"]` | RouteValue（友好 URL） |
| `txtUser.Text` / `ddlRole.SelectedValue` | Form（控件 `name=ctl00$...$txtUser`） |
| `FileUpload1.PostedFile` | Form / multipart |

控件参数命名规则：实际表单字段名为 `ctl00$ContentPlaceHolder$txtUser` 形式，下游审计时需还原。

## 6. .ashx HttpHandler

```csharp
public class Upload : IHttpHandler
{
    public void ProcessRequest(HttpContext context)
    {
        var name = context.Request.Form["name"];
        // ...
    }
    public bool IsReusable => false;
}
```

提取规则：
- URL = `.ashx` 物理路径
- 入口固定为 `ProcessRequest(HttpContext)`
- 参数全部从 `context.Request` 取，按 §5 规则归类

## 7. .asmx Web Service

```csharp
[WebService(Namespace = "http://tempuri.org/")]
public class UserSvc : WebService
{
    [WebMethod]
    public string Login(string user, string pass) { ... }
}
```

提取规则：
- 每个 `[WebMethod]` 标注的方法为一个端点
- URL = `/UserSvc.asmx/Login`
- 参数 = 方法形参，可用 SOAP / GET / POST 三种方式调用
- 输出多条记录，分别对应 SOAP 和 HTTP POST 形态

## 8. web.config Handler 注册

```xml
<system.webServer>
  <handlers>
    <add name="UploadHandler" verb="POST" path="Upload.do"
         type="MyApp.UploadHandler, MyApp" />
  </handlers>
</system.webServer>
```

提取规则：
- 每条 `<add>` 一个端点
- `verb` → HTTP 方法
- `path` → URL（支持通配符 `*.do`）
- `type` → `命名空间.类名, 程序集名` → 反编译该程序集的对应类（必须实现 `IHttpHandler`）

## 9. 关键搜索词

```
<%@ Page  /  <%@ Control  /  <%@ MasterType  /  <%@ WebHandler  /  <%@ WebService
Inherits="
MapPageRoute(
RouteTable.Routes
Page_Load / OnInit / OnLoad / OnPreRender
Request.QueryString / Request.Form / Request.Cookies / Request.Headers / Request[
Page.RouteData
[WebMethod]
IHttpHandler
ProcessRequest(
<httpHandlers / <handlers
```

## 10. 鉴权来源

Web Forms 通常通过 `web.config` 配置：
```xml
<location path="admin">
  <system.web>
    <authorization>
      <deny users="?" />
      <allow roles="Admin" />
    </authorization>
  </system.web>
</location>
```

输出每条路由时必须根据 `<location path>` 规则匹配并标注鉴权状态。
