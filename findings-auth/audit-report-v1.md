# 安全审计报告 v1（对象：qtcloud-auth @ f017ffa）

> 审计方：量潮安全工程实验室（quanttide-laboratory-of-security-engineering）
> 审计日期：2026-08-16 ｜ 对象基线：`quanttide/qtcloud-auth` @ `f017ffa`
> 方法：静态代码审计 + 威胁建模 + 基础设施审计 + 6 层工具链扫描（PoC-001 矩阵复用）
> 配套：漏洞台账 `vulnerability-ledger-v1.md`、复盘报告 `review-report-v1.md`

## 1. 执行摘要

对量潮身份云（qtcloud-auth）的审计发现 **16 项问题（2 High / 6 Medium / 6 Low / 2 Info）**，整体结论：

- **密码学基础扎实**：Argon2id（OWASP 参数 + 常数时间比较）、JWT 验签无算法混淆、gorm 参数化查询、token store DB 共享——均达标；
- **最高风险是「配置守卫缺失」与「认证爆破面」**：代码默认值（JWT 密钥/管理员密码/console SMS）无 prod 环境拒绝守卫（对比 qtcloud-secret 已修复的 R2 模式）；密码登录与 SMS 验证码均无限流——**身份服务最核心的攻击面未设防**；
- **依赖面健康**：无第三方高危漏洞（jwt v5.3.0 干净），优于 qtcloud-secret 的 jwt 5.2.1；
- **仓库卫生良好**：无 .gocache 入库；gitleaks 仅 4 条 docs 示例（非真实密钥）。

## 2. 范围与方法

| 维度 | 内容 |
|------|------|
| 代码面 | provider 全部 Go 文件（oauth/password/sms/admin/middleware/handler/store/model/app/main） |
| 基础设施面 | `manifests/terraform/*.tf`（FC/RDS/VPC/state） |
| 供应链面 | `.github/workflows/deploy-provider.yml`、go.mod 依赖 |
| 方法 | 手工审计 + 6 层工具链（govulncheck/osv-scanner/gosec/gitleaks/tfsec/trivy） |

## 3. 发现明细（按严重级）

| ID | 严重级 | 标题 |
|----|--------|------|
| A-01 | High | 无环境守卫：默认 JWT 密钥 / console SMS / 测试码静默生效 |
| A-02 | High | 密码登录无速率限制/账户锁定（在线爆破面） |
| A-03 | Medium | SMS 验证码无尝试次数限制（暴力面） |
| A-04 | Medium | SMS 验证码写入日志（console 驱动为默认） |
| A-05 | Medium | 种子管理员默认密码 "123456" |
| A-06 | Medium | RDS 连接 sslmode=disable + 凭据明文入 tfstate |
| A-07 | Medium | 无 token 吊销 + refresh 无轮换 |
| A-08 | Medium | O(n) 全表扫描（登录/验证码/限流判断） |
| A-09 | Low | ADMIN_TOKEN 非恒定时间比较 |
| A-10 | Low | http.Server 无超时（Slowloris） |
| A-11 | Low | Argon2id 参数来自存储（内存炸弹面） |
| A-12 | Low | 注册密码策略弱（仅 6 位下限） |
| A-13 | Low | JWT 签发仍 HS256（RS256 升级未落地） |
| A-14 | Low | 未处理错误（gosec G104） |
| A-15 | Info | docs 示例 JWT 无标注 |
| A-16 | Info | CI 供应链硬化 |
| D-01 | Medium | Go 工具链 stdlib 漏洞集（33 条，浮动 tag） |
| D-02 | Info | x/crypto openpgp 通告（不适用，已接受） |

## 4. 威胁建模（STRIDE × 资产）结论

| 资产 | 主要风险 | 状态 |
|------|----------|------|
| 用户凭证（密码哈希） | 泄露：爆破（A-02/A-05 叠加）→ 账户接管 | **未设防（最高优先）** |
| JWT 访问令牌 | 仿冒：HS256 对称密钥（A-13）+ 默认密钥（A-01）→ 任意用户伪造 | 未修复 |
| SMS 验证码 | 绕过：无尝试限制（A-03）+ 日志泄露（A-04）→ 任意手机号登录 | 未修复 |
| 会话（refresh token） | 否认/长期占用：无吊销（A-07） | 未修复 |
| 用户数据（RDS） | 泄露：tfstate 明文凭据（A-06）→ 数据库全量泄露 | 部分缓解（VPC 内） |
| 管理员端点 | 提权：弱口令种子（A-05）+ 时序比较（A-09） | fail-closed ✓ 但口令风险在 |

## 5. 与 PoC-001（qtcloud-secret）对照

| 维度 | qtcloud-secret | qtcloud-auth | 结论 |
|------|----------------|--------------|------|
| 依赖漏洞 | jwt 5.2.1 有 CVE（High） | 依赖干净 | auth 更优 |
| 默认密钥守卫 | 已修复（ENV=prod 拒绝） | **缺失** | auth 需补 |
| 授权模型 | 无粒度（High） | 单角色（admin/普通），粒度同样有限 | 同类问题 |
| 限流 | 无（Medium） | 无（密码 High） | 身份服务更严重 |
| 验证码 | 不适用 | 有实现但无尝试限制 | auth 专项 |
| 仓库卫生 | .gocache 入库 | 干净 | auth 更优 |
| 密码哈希 | bcrypt 历史/AES 加密 | Argon2id | auth 更优 |

## 6. 整改建议清单

| 优先级 | 事项 | 关联 |
|--------|------|------|
| **P0（立即）** | ENV=prod 守卫：拒绝默认 JWT_SECRET / console SMS / SMS_TEST_CODE / 默认 ADMIN_PASSWORD（A-01/A-04/A-05） | 配置面根因 |
| P0 | 密码登录限流 + 账户锁定（A-02） | 核心爆破面 |
| P1（短期） | SMS 码尝试上限（A-03）；token 吊销端点 + refresh 轮换（A-07）；RDS sslmode + 凭据迁移配置中心（A-06）；索引直查替代全表扫描（A-08） | 认证/数据面 |
| P2（中期） | ADMIN_TOKEN 常数时间（A-09）；Server 超时（A-10）；Argon2id 参数上限（A-11）；密码策略（A-12）；RS256 签发落地（A-13）；错误处理（A-14）；CI 硬化（A-16） | 纵深防御 |
| 持续 | Go 工具链升级 + 镜像固定 digest（D-01）；季度巡检 | 依赖面 |

## 7. 残余风险与结论

1. **身份服务是安全云生态的信任根**：A-01/A-02 两项 High 应作为最高优先级修复——它们直接决定「能否伪造身份」与「能否爆破进入」；
2. 单团队阶段的授权模型（admin/普通用户）可接受，多团队阶段需与 qtcloud-secret 的权限治理一起规划（安全云「身份与访问安全」能力域输入）；
3. 本报告与台账模板为 PoC-001 模板的直接复用，**方法可复用性已验证**（见复盘报告）。
