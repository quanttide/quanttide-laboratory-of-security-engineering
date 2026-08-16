# quanttide-laboratory-of-security-engineering

量潮安全工程实验室

## 实验索引

### PoC-001：漏洞管理与安全审计（对象 qtcloud-secret）✅ 已完成

以量潮机密云（qtcloud-secret @ `cec7701`）为审计对象，验证量潮安全云首发能力域——**漏洞管理**与**安全审计**——的两条完整链路，沉淀可复用的工具链、审计报告模板与漏洞台账模板。

- 规划文档：[docs/poc-vuln-audit-qtcloud-secret.md](docs/poc-vuln-audit-qtcloud-secret.md)
- 扫描产物：`scans/`（8 层扫描矩阵原始报告）
- 审计产物：`findings/`（漏洞台账 v1、审计报告 v1、复盘报告 v1）
- 结论摘要：15 条分级发现（2 High / 5 Medium / 7 Low / 1 Info）；R2 默认密钥后门验证已修复；最高风险为授权模型（A-01/R3）

### PoC-002：漏洞管理与安全审计（对象 qtcloud-auth）✅ 已完成

以量潮身份云（qtcloud-auth @ `f017ffa`）为审计对象，验证 PoC-001 方法在身份云的可复用性，覆盖密码哈希 / JWT / OAuth2 / SMS / ADMIN_TOKEN / RDS 六个安全关键面。

- 规划文档：[docs/poc-vuln-audit-qtcloud-auth.md](docs/poc-vuln-audit-qtcloud-auth.md)
- 扫描产物：`scans-auth/`（6 层扫描矩阵原始报告）
- 审计产物：`findings-auth/`（漏洞台账 v1、审计报告 v1、复盘报告 v1）
- 结论摘要：18 条发现（2 High / 6 Medium / 6 Low / 2 Info + 依赖 2 条）；最高风险为配置守卫缺失（A-01）与认证爆破面（A-02）；依赖面健康、Argon2id 合规

## 目录结构

```
examples/default/
├── README.md                                     # 本文件（实验索引）
├── docs/
│   ├── poc-vuln-audit-qtcloud-secret.md          # PoC-001 规划（已完成）
│   └── poc-vuln-audit-qtcloud-auth.md            # PoC-002 规划（已完成）
├── scans/                                        # PoC-001 扫描原始报告
│   ├── 01-govulncheck.json … 08-dart-analyze.json
├── scans-auth/                                   # PoC-002 扫描原始报告
│   ├── 01-govulncheck.json … 06-trivy-fs.json
├── findings/                                     # PoC-001 审计产物
│   ├── vulnerability-ledger-v1.md
│   ├── audit-report-v1.md
│   └── review-report-v1.md
└── findings-auth/                                # PoC-002 审计产物
    ├── vulnerability-ledger-v1.md
    ├── audit-report-v1.md
    └── review-report-v1.md
```
