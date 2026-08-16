# 复盘报告 v1（PoC-002：漏洞管理与安全审计，对象 qtcloud-auth）

> 实验室：quanttide-laboratory-of-security-engineering ｜ 日期：2026-08-16
> PoC 规划：`docs/poc-vuln-audit-qtcloud-auth.md` ｜ 配套：漏洞台账 v1、审计报告 v1

## 1. PoC 目标达成度

| 成功标准 | 达成 | 说明 |
|----------|------|------|
| 扫描矩阵在第二对象复跑 | ✅ | 6 层矩阵全部产出（无 studio → dart analyze 跳过；semgrep 环境限制同 PoC-001） |
| 台账覆盖全部扫描层 | ✅ | `vulnerability-ledger-v1.md`：18 条发现（2 High / 6 Medium / 6 Low / 2 Info + 2 依赖） |
| 身份云重点面全覆盖 | ✅ | 密码哈希 / JWT / OAuth2 / SMS / ADMIN_TOKEN / RDS 六面逐文件取证 |
| 模板复用 PoC-001 | ✅ | 台账/报告/复盘三模板直接复用，零结构调整 |

## 2. 关键发现（身份云专项）

1. **配置守卫缺失是身份云与机密云的最大差异**：qtcloud-secret 已修复 R2（prod 拒绝 fallback），qtcloud-auth 完全无 ENV 概念——默认 JWT 密钥、默认管理员密码、console SMS 驱动、固定测试码四处默认值静默生效；
2. **爆破面集中**：密码登录 + SMS 验证码均无限流，且 O(n) 全表扫描放大每次尝试成本——三个 Medium/High 叠加构成「认证暴力破解」完整链条；
3. **对称签名未跟进**：qtcloud-secret 已按 RS256 验签，qtcloud-auth 签发侧仍 HS256——跨仓库安全架构不同步；
4. **正面差异**：依赖干净（jwt 5.3.0）、仓库无 .gocache、Argon2id 合规——证明审计既发现问题也确认质量。

## 3. 方法学复盘（PoC-001 → PoC-002）

| 维度 | PoC-001 经验 | PoC-002 验证 |
|------|--------------|--------------|
| 扫描矩阵 | 8 层全量 | 6 层适配（无 studio/dart 层）——矩阵可按对象裁剪 ✓ |
| 工具误报 | gitleaks .gocache 夹具需人工判定 | docs 示例 JWT 同理人工判定 ✓ |
| 手工审计重点 | 按对象架构定制（secret：加密/导出） | 按身份面定制（auth：凭证/令牌/验证码）✓ |
| govulncheck 解析 | JSON 流式解析脚本 | 直接复用 ✓ |
| semgrep | 环境受限 | 同样受限（记录在案，待 CI 补跑） |

**结论：PoC-001 的方法与模板在第二个对象上完整复用成功，未做结构性调整。**

## 4. 差距清单（对象仓库 → 安全云能力）

| 能力域 | 输入 |
|--------|------|
| 身份与访问安全 | A-01/A-02/A-13 直接定义「认证加固」产品需求（环境守卫、限流、非对称签名） |
| 漏洞管理 | D-01 工具链巡检机制；A-08 性能面提示 |
| 安全运营 | A-07 会话吊销 → token 生命周期管理功能 |

## 5. 后续建议

1. **对象仓库（qtcloud-auth）**：16 条发现已提为 GitHub issues（`#1`–`#16`，标签 `poc-002` + `security:*`，https://github.com/quanttide/qtcloud-auth/issues），处理人已分配 **@LiXiang050789**。按 P0（A-01/A-02/A-04/A-05 配置守卫 + 限流、D-01 工具链）→ P1（A-03/A-06/A-07/A-08）→ P2 顺序整改；issue 关闭即台账状态机「复测中 → 已关闭」的触发点（D-02、A-15 已接受项未提 issue）；
2. **跨仓库协同**：qtcloud-auth 的 RS256 签发（A-13）与 qtcloud-secret 的 RS256 验签需同步落地，避免长期不对称；
3. **实验室**：PoC 模板已稳定，可并行推进第三个对象（qtcloud-devops 供应链面 或 qtcloud-pay 支付面）。

## 6. 结论

PoC-002 达成全部目标：**验证了审计方法在身份云对象的可复用性**，产出 18 条分级发现（2 High / 6 Medium / 6 Low / 2 Info + 依赖 2 条），确认了「配置守卫缺失 + 爆破面未设防」这一身份服务最紧迫风险，模板零调整复用。
