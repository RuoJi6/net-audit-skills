# IDOR / 水平越权检查清单

当接口接收或使用资源标识时读取本文件。目标是确认“已登录用户是否只能访问自己有权访问的资源”。

## 高风险参数名

优先追踪以下参数、路由变量、Body 字段、Query/Form 字段：

```text
userId
uid
accountId
ownerId
memberId
tenantId
orgId
deptId
departmentId
companyId
projectId
orderId
fileId
docId
roleId
groupId
customerId
```

## 必查证据

- 当前用户身份来源：`User.Identity`、`ClaimsPrincipal`、`Session`、Cookie、JWT Claims、自定义上下文。
- 资源查询条件是否绑定当前用户、租户、部门或组织。
- 服务层是否二次校验 owner/member/tenant 关系。
- 数据访问层是否只按传入 ID 查询，未加归属条件。
- 管理员角色是否有明确分支，普通用户是否会进入同一分支。
- 批量查询、导出、下载、删除、更新接口是否做逐项归属校验。

## 判定规则

- 仅有 `[Authorize]` 或登录态校验：最多证明“已认证”，不能证明“资源授权完整”。
- 查询条件包含 `id = request.Id` 但没有当前用户/租户约束：标记为 IDOR 可疑。
- 资源读取后没有校验 `resource.OwnerId == currentUserId` 或等价关系：标记为 IDOR 可疑。
- 权限校验只在前端菜单或按钮中出现：不能作为后端授权证据。
- 管理员接口可以跳过资源归属，但必须有可靠角色/Policy 证据。
- 复杂 RBAC/ABAC 无法静态确认时，标为“待验证”，列出缺失证据。

## 证据链模板

```text
入口: Controller.Action / Page / Handler
资源标识: userId / tenantId / fileId / ...
身份来源: User.Claims / Session["UserId"] / ...
查询路径: Controller -> Service -> Repository
已发现校验: 登录 / 角色 / 资源归属 / 无
缺失校验: owner / tenant / member / dept
结论: 已验证 / 待验证 / 不成立
```

## 常见失败模式

- 只检查 Controller，没追到 Service/Repository。
- 看到 `IsAuthenticated` 就认定安全。
- 忽略导出、下载、批量删除等非典型 CRUD 接口。
- 没区分“传入 userId 与当前用户一致”校验和“任意 userId 查询”。
- 没检查租户隔离，导致跨公司/跨部门访问漏报。
