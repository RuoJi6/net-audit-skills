---
name: net-auth-audit
description: .NET / ASP.NET 鉴权机制审计。用于用户要求检查登录校验、权限校验、[Authorize]/[AllowAnonymous]/Role/Policy、未授权访问、越权/IDOR、基于路由映射分析接口鉴权状态，或对只有 bin/dll/web.config/aspx 的闭源项目先反编译再审计鉴权。适用于 ASP.NET Core、MVC5/Web API2、Web Forms、Minimal API、Handler/ASMX；不用于单纯路由提取、通用漏洞审计、鉴权原理解释、登录功能开发、前端菜单权限或 NuGet CVE 检查。
---

# .NET 鉴权机制审计

从 .NET / ASP.NET 项目源码、反编译产物或 `net-route-mapper` 输出中梳理鉴权机制，覆盖所有路由、页面和接口，识别鉴权绕过、未授权访问、权限缺失和资源级越权问题，并按固定目录结构生成 Markdown 审计产物。

## 适用与不适用

适用：
- 审计 ASP.NET Core / MVC5 / Web API2 / Web Forms / Minimal API / Handler / ASMX 的鉴权机制
- 检查登录态、角色、Policy、Claim、自定义权限 Attribute、Filter、Middleware、BaseController、Page 基类
- 基于路由清单或源码输出每个接口的鉴权状态
- 闭源项目只有 `bin/*.dll`、`.aspx`、`web.config` 时，先复用或生成共享反编译产物再审计
- 检查水平越权 / IDOR / 租户隔离 / 资源归属校验

不适用：
- 只提取路由和参数：使用 `net-route-mapper`
- SQL 注入、XSS、SSRF、文件上传、文件读取等专项审计
- 只解释 JWT / OAuth / Cookie / Session 原理
- 帮用户实现登录、注册、权限系统
- 只审计前端菜单、按钮显隐、前端路由守卫
- 只做代码质量、架构优化或通用 NuGet/CVE 依赖检查

## 核心规则

- 必须先梳理项目鉴权机制，再下漏洞结论。
- 必须覆盖所有路由、页面、接口；不得只列高风险接口，不得用“等”“其他”省略。
- 每个结论必须有代码证据或配置证据，标注文件、类/方法、配置节或反编译产物路径。
- 必须区分认证、角色/策略授权、资源级授权；“需要登录”不等于“权限完整”。
- 禁止只凭 Action 缺少 `[Authorize]` 判定未授权，必须检查全局 Filter、FallbackPolicy、DefaultPolicy、中间件、基类、路由组和 `web.config`。
- 禁止把前端菜单、按钮隐藏、前端路由守卫当作后端授权证据。
- 禁止执行破坏性请求。PoC 只能给非破坏性验证思路或 HTTP 请求模板。
- 高风险但证据不完整时标为“待验证”，不得编造漏洞。
- 本 Skill 不做鉴权组件版本/CVE 审计，除非后续明确扩展。

## 工作流

### 1. 初始化输入

1. 确认输入类型：源码项目、反编译产物、`net-route-mapper` 输出、或混合输入。
2. 若已有 `net-route-mapper` 输出，优先作为路由基线；否则自行枚举路由和页面，不得跳过。
3. 识别项目类型：ASP.NET Core、MVC5/Web API2、Web Forms、Minimal API、Handler/ASMX、混合项目。
4. 读取 `references/AUTH_PATTERNS.md`，按项目类型定位鉴权入口和搜索词。

### 2. 处理闭源或缺源码项目

若源码缺失，或 `.aspx` / 配置指向的后端类只存在于 dll：

1. 优先检查 `{project}_audit/decompiled/manifest.json`。
2. `status=success` 且 `input_dir` 与当前目标匹配：直接复用 `{project}_audit/decompiled/`。
3. `status=partial`：可复用成功程序集，但必须在 README 和主报告中标注失败项及审计不完整范围。
4. 入口程序集失败、manifest 缺失、不匹配或不可用：按 `../net-route-mapper/references/DECOMPILE_STRATEGY.md` 执行共享二进制预处理。
5. 若存在 `{project}_audit/deobfuscated/manifest.json`，README 必须记录其状态；反编译应优先基于 `deobfuscated/bin_clean/`。

不要在本 Skill 中维护另一套反编译规范；共享预处理产物是项目级审计证据，供多个 net audit skills 复用。

### 3. 梳理鉴权机制

必须输出以下机制结论：
- 认证方式：Cookie / Session / FormsAuthentication / Windows / JWT Bearer / 自定义 Token / 混合
- 授权方式：Role / Policy / Claim / 自定义权限服务 / 页面或控制器基类 / Filter / Middleware
- 全局规则：FallbackPolicy、DefaultPolicy、GlobalFilters、`web.config <authorization>`、`location path`
- 白名单与匿名入口：`[AllowAnonymous]`、`AllowAnonymous()`、`permitAll` 等价逻辑、登录页、静态资源、健康检查、公开 API
- 资源级授权：用户、租户、部门、公司、owner/member 关系是否在服务层或查询条件中约束

### 4. 生成鉴权映射

对每个路由、页面或接口给出一行映射：
- 路由/页面、HTTP 方法、入口位置
- 鉴权状态：公开、受保护、仅认证、无鉴权、不确定
- 鉴权证据：Attribute、Filter、Middleware、web.config、基类、代码判断
- 所需角色/Policy/Claim
- 资源级授权状态
- 风险与备注

状态不确定时必须说明缺失证据，不得省略该接口。

### 5. 漏洞分析

只有满足证据链时才写入主报告漏洞详情：
- 可达入口明确
- 相关鉴权层级已检查
- 后续拦截层是否存在已分析
- 绕过后影响范围明确
- 资源归属缺失或权限缺失有代码证据

IDOR / 水平越权必须读取 `references/IDOR_CHECKLIST.md`。

漏洞编号和评分必须读取 `../net-shared/SEVERITY_RATING.md`，沿用 `{C/H/M/L}-AUTH-{序号}`，例如 `H-AUTH-001`。每个漏洞必须给出 R/I/C、Score/CVSS 和判定理由。

### 6. 固定输出目录

审计完成必须按 `net-route-mapper` 的风格生成固定目录和固定文件名，不使用时间戳文件名：

```text
{project}_audit/auth_audit/
├── README.md                 # 主索引：范围、机制概览、风险统计、产物链接
├── auth_mapping.md           # 全部路由/页面/接口的鉴权状态表
├── auth_findings.md          # 漏洞详情、证据链、PoC、修复建议
└── modules/
    ├── {ModuleA}.md          # 每模块详细鉴权证据和备注
    ├── {ModuleB}.md
    └── ...
```

模板：
- 主索引：`references/OUTPUT_TEMPLATE_README.md` → 输出为 `README.md`
- 鉴权映射表：`references/OUTPUT_TEMPLATE_MAPPING.md` → 输出为 `auth_mapping.md`
- 漏洞详情：`references/OUTPUT_TEMPLATE_MAIN.md` → 输出为 `auth_findings.md`

职责划分：
- `README.md` 记录范围、输入、工具、共享二进制 manifest、机制概览、风险统计和产物链接；不要放完整漏洞详情。
- `auth_mapping.md` 必须列出所有路由/页面/接口；不要放漏洞详细分析。
- `auth_findings.md` 只放漏洞详情、证据链、PoC、修复建议；不要放完整路由清单。
- `modules/` 用于大型项目按模块拆分鉴权证据；小项目可不生成模块文件，但目录结构应保持一致。

## Gotchas

- ASP.NET Core 不再使用 `web.config` 配置应用鉴权，重点看 `Program.cs` / `Startup.cs` / `appsettings.json` / 中间件顺序。
- `UseAuthentication()` 必须在 `UseAuthorization()` 前；顺序异常可能导致鉴权失效或行为偏差。
- `FallbackPolicy` / `DefaultPolicy` 可能让无 `[Authorize]` 的接口默认受保护。
- MVC5 / Web API2 可能通过 `GlobalFilters.Filters.Add(new AuthorizeAttribute())` 全局鉴权。
- Web Forms 常把鉴权放在 Page 基类、`OnInit`、`Page_Load`、母版页或 `web.config location path` 中。
- `OnActionExecuting` / `IAsyncAuthorizationFilter` / 自定义 Attribute 可能承担真实权限校验。
- `[AllowAnonymous]` 不必然是漏洞；必须结合登录页、公开资源、健康检查、业务公开接口判断。
- `.ashx`、`.asmx`、Razor Pages、Minimal API、MapGroup 路由容易漏。
- 反编译后不要只看命名清晰的类；混淆类、事件处理器和基类也可能包含鉴权逻辑。
- 只检查登录态会漏掉 IDOR；凡出现 `userId`、`tenantId`、`deptId`、`companyId`、`ownerId` 等资源标识，都要追踪归属校验。

## 完成前自检

- [ ] 已识别项目类型和输入类型
- [ ] 已复用或生成共享反编译 manifest（闭源/缺源码场景）
- [ ] 已覆盖所有路由、页面、接口
- [ ] 已梳理认证、授权、白名单、全局规则、资源级授权
- [ ] 每个接口都有鉴权状态和证据
- [ ] 漏洞结论都有代码或配置证据
- [ ] 未把前端权限当作后端授权证据
- [ ] 已按 `{project}_audit/auth_audit/` 固定目录生成 `README.md`、`auth_mapping.md`、`auth_findings.md` 并互相链接

## 维护评测

触发、不触发、边界与失败案例维护在 `references/EVAL_CASES.md`。修改 description、工作流或输出模板后，必须回看这些样例，确保触发边界没有变宽或变窄。
