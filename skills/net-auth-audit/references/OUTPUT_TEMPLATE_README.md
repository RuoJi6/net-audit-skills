# 鉴权审计说明文档模板

> 硬约束：
> 1. README 是 `auth_audit/` 目录主索引，记录审计范围、机制概览、风险统计、输入、方法、工具、局限性和产物链接。
> 2. 不写具体漏洞详情，具体漏洞放 `auth_findings.md`。
> 3. 输出文件固定为 `{project}_audit/auth_audit/README.md`。

# 【项目名称】 - 鉴权审计说明

生成时间: 【YYYY-MM-DD HH:MM:SS】
分析路径: 【项目路径】

## 1. 审计范围

| 项 | 内容 |
| --- | --- |
| 项目类型 | 【ASP.NET Core / MVC5 / Web API2 / Web Forms / Minimal API / 混合】 |
| 输入来源 | 【源码 / route-mapper 输出 / 反编译产物 / 混合】 |
| 覆盖范围 | 【路由、页面、Handler、ASMX、Minimal API 等】 |
| 不在范围 | 通用 NuGet/CVE、SQL/XSS/SSRF/上传、前端菜单权限、代码质量 |

## 2. 二进制预处理记录

| 项 | 状态 | 路径/说明 |
| --- | --- | --- |
| 源码是否可用 | 【是/否/部分】 | 【说明】 |
| 反混淆 manifest | 【无/success/partial/failed/skipped】 | `{project}_audit/deobfuscated/manifest.json` |
| 反编译 manifest | 【无/success/partial/failed/skipped】 | `{project}_audit/decompiled/manifest.json` |
| 反编译产物 | 【已复用/新生成/未使用】 | 【路径】 |
| 不完整范围 | 【无/程序集或入口列表】 | 【说明】 |

说明：闭源或缺源码场景必须复用或生成共享二进制预处理产物，流程参考 `skills/net-route-mapper/references/DECOMPILE_STRATEGY.md`。

## 3. 审计方法

- 识别项目类型和鉴权入口。
- 枚举所有路由、页面、接口。
- 梳理认证、角色/Policy/Claim、白名单、全局规则和资源级授权。
- 对每个入口建立“接口 -> 鉴权证据 -> 结论”的链路。
- 对涉及资源标识的接口执行 IDOR / 水平越权检查。
- 对确认的问题按 `../net-shared/SEVERITY_RATING.md` 进行 `{C/H/M/L}-AUTH-{序号}` 编号和 R/I/C 评分。
- 对证据不足的结论标注“不确定”或“待验证”。

## 4. 使用的关键搜索面

| 类型 | 搜索面 |
| --- | --- |
| ASP.NET Core | `Program.cs`、`Startup.cs`、`UseAuthentication`、`UseAuthorization`、`FallbackPolicy`、`RequireAuthorization` |
| MVC5 / Web API2 | `GlobalFilters`、`FilterConfig`、`WebApiConfig`、`AuthorizeAttribute`、`OnActionExecuting` |
| Web Forms | `web.config`、`location path`、`FormsAuthentication`、`Inherits`、`OnInit`、`Page_Load` |
| 资源授权 | `userId`、`tenantId`、`deptId`、`companyId`、`ownerId`、Service/Repository 查询条件 |

## 5. 局限性

- 静态审计无法完全替代动态验证。
- 反编译失败或入口程序集缺失会影响覆盖率。
- 混淆代码可能导致类名、方法名和控制流可读性下降。
- 复杂业务权限需要结合业务规则确认。
- 前端权限只作为线索，不作为后端授权证据。

## 6. 验证建议

- 对“无鉴权”“仅认证”“资源级授权缺失”的接口进行非破坏性动态验证。
- 分别使用未登录用户、普通用户、低权限用户和管理员用户验证。
- 对 IDOR 可疑接口使用同租户不同用户、跨租户用户进行对比验证。
- 动态验证前确认测试环境和授权边界，避免破坏生产数据。

## 7. 关联文档

- 路由-鉴权映射表: [auth_mapping.md](auth_mapping.md)
- 鉴权风险详情: [auth_findings.md](auth_findings.md)
- 模块详情目录: [modules/](modules/)

## 输出自检

- [ ] 已记录输入来源和审计范围
- [ ] 已记录反混淆/反编译 manifest 状态
- [ ] 已说明局限性和验证建议
- [ ] README 不包含具体漏洞详情
- [ ] 已链接 `auth_mapping.md`、`auth_findings.md` 和 `modules/`
