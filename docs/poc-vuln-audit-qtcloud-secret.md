# PoC 规划：漏洞管理与安全审计（对象：qtcloud-secret）

> 实验室：`quanttide-laboratory-of-security-engineering`（量潮安全工程实验室）
> 状态：**已执行完成**（2026-08-16）。审计产物见 `../findings/`（漏洞台账 v1、审计报告 v1、复盘报告 v1）
> 关联：量潮安全云首发能力域——漏洞管理、安全审计（见 `qtcloud-security` README）

## 1. 背景与目标

量潮安全云（qtcloud-security）首发能力域为**漏洞管理**与**安全审计**。实验室（内部结构自由，不受领域仓库约束）是能力验证的试验田：本 PoC 以最邻近的成熟应用 **qtcloud-secret（量潮机密云）** 为审计对象，跑通两条能力链路：

- **漏洞管理闭环**：扫描 → 登记 → 评级 → 跟踪 → 修复 → 复测
- **安全审计闭环**：审计 → 取证 → 报告 → 整改 → 残余风险

目标产出不止于「发现问题」，而是沉淀**可复用的工具链、报告模板、漏洞台账模板**，作为安全云产品化的直接输入。

### 成功标准

1. 全链路扫描**一键复跑**（脚本化，无需真实凭证）；
2. 漏洞台账 ≥ 1 份，覆盖全部扫描层面，每条含 CVSS/EPSS 评级与跟踪状态；
3. 审计报告对 qtcloud-secret 已知风险清单 **R1–R11 逐条给出代码级证据**（已验证 / 已缓解 / 待整改）；
4. 审计报告模板与漏洞台账模板可复用（后续安全云对外审计直接沿用）。

## 2. 审计对象：qtcloud-secret @ `cec7701`

> 基线锁定：PoC 全程以 commit `cec7701`（main）为审计对象，防止对象演进导致基线漂移。

### 2.1 架构速览

```
Flutter Studio（登录后明文交互）
      │  JWT（qtcloud-auth 签发）
      ▼
FC 应用服务（Go provider）──MASTER_KEY（AES-256-GCM）──▶ OSS（密文落盘 + SSE-OSS + 私有 ACL）
      │
      └── Terraform（远程 state 桶 + FC + RDS 规划）── CI（tag 触发部署，org secret 注入）
```

信任模型：**服务端可信**——MASTER_KEY 由运维管理，明文存在于 FC 环境变量 / tfstate / GitHub org secret 三处（R1）。

### 2.2 攻击面清单

| 层面 | 范围 | 关键文件/资源 |
|------|------|----------------|
| 代码面（服务端） | Go provider | `cmd/server/main.go`、`internal/auth/jwt.go`、`internal/config/config.go`、`internal/crypto/crypto.go`、`internal/handler/{cors,secrets}.go`、`internal/model/item.go`、`internal/storage/{oss,storage}.go` |
| 客户端面 | Flutter studio | token 内存态实践、明文可见面、Web 产物逆向（R10/R11） |
| 基础设施面 | Terraform IaC | `manifests/terraform/*.tf`（OSS 桶/ACL/SSE、FC、state、CDN） |
| 供应链面 | CI/CD | `.github/workflows/deploy-{provider,studio}.yml`（tag 触发、org secret 注入链） |
| 依赖面 | Go + Dart | `src/provider/go.mod`、`src/studio/pubspec.lock` |
| 流程面 | 运维制度 | 主密钥轮换、备份导出（明文 NDJSON）、审计日志（容器 stdout） |

### 2.3 已知风险基线（security.md R1–R11）

| 风险 | 等级 | 内容摘要 | 审计关注点 |
|------|------|----------|------------|
| R1 | 🔴 高 | MASTER_KEY 单点，明文见三处 | tfstate/FC 环境变量/org secret 可见面 |
| R2 | 🔴 高 | 默认 JWT 密钥 fallback 后门 | **已修复**（`28cdc60`，ENV=prod 拒绝启动）；验证修复有效性 |
| R3 | 🔴 高 | 账号 = 全量数据（无细粒度权限） | handler 层越权审计 |
| R4 | 🟠 中 | 会话 token 无撤销 | auth 侧依赖评估 |
| R5 | 🟠 中 | 无应用层速率限制 | 登录/API 限流缺失 |
| R6 | 🟠 中 | 审计不足（stdout 日志） | 审计落盘与告警缺失 |
| R7 | 🟠 中 | 备份为明文副本 | 导出接口权限与加密 |
| R8–R11 | 🟡 低 | 元数据明文、版本保留、客户端明文面、逆向 | 改进项核对 |

## 3. 漏洞管理 PoC

### 3.1 扫描矩阵

| 层面 | 工具 | 扫描目标 | 输出 |
|------|------|----------|------|
| Go 依赖漏洞 | `govulncheck` | `src/provider` | 依赖漏洞清单（OSV） |
| 全依赖匹配 | `osv-scanner` | go.mod / pubspec.lock | SBOM 级漏洞匹配 |
| Go 静态分析 | `gosec` | `src/provider` | 静态告警（含 CWE 映射） |
| 语义规则 | `semgrep` | Go + Dart 源码 | 自定义规则匹配（硬编码密钥、弱加密、危险函数） |
| Dart 依赖 | `dart pub outdated` / pub.dev advisory | `src/studio` | Flutter 侧依赖漏洞 |
| 容器镜像 | `trivy` | provider Dockerfile 构建产物 | 镜像 CVE |
| IaC | `tfsec` / `checkov` | `manifests/terraform` | 基础设施风险（桶 ACL、state、权限） |
| 密钥泄露 | `gitleaks` | 全仓库历史 | 硬编码密钥/凭据（对照 R2） |

### 3.2 漏洞管理闭环（演示安全云能力）

```
扫描结果 ─登记→ 漏洞台账 ─评级→ CVSS 3.1 + EPSS ─指派→ 状态机跟踪 ─修复→ 复测 ─关闭→ 复盘
```

漏洞台账字段：`ID / 来源层 / 工具 / CVE(OSV) / 受影响组件与版本 / CVSS 3.1 / EPSS / 利用条件 / 严重级 / 状态 / 负责人 / 发现日期 / 修复日期 / 复测结果`。

状态机：`新发现 → 分析中 → 待修复 → 修复中 → 复测中 → 已关闭`（`已接受` 用于低风险残余项）。

### 3.3 评级方法

- 基础分：CVSS 3.1（漏洞库直接给出）；未入库的代码级发现按 CVSS 3.1 规范人工评分；
- 修正：EPSS 概率修正优先级；结合本对象实际利用条件（如 R2 类硬编码密钥 → 直接判定 Critical 且优先复测）；
- 输出：严重级（Critical / High / Medium / Low）+ 排序后的整改队列。

## 4. 安全审计 PoC

### 4.1 代码审计重点（对照 R1–R11 逐条取证）

| 文件 | 审计重点 | 关联风险 |
|------|----------|----------|
| `internal/crypto/crypto.go` | AES-256-GCM 实现正确性、nonce 随机性与唯一性、密钥来源 | R1 |
| `internal/auth/jwt.go` | 验签实现、公钥来源、fallback 路径是否彻底移除 | R2 |
| `internal/config/config.go` | 默认值注入、ENV=prod 校验逻辑 | R2 |
| `internal/handler/secrets.go` | 越权（无细粒度权限）、批量导出、输入校验 | R3/R7 |
| `internal/handler/cors.go` | CORS 配置、来源校验 | R3/R10 |
| `internal/storage/oss.go` | 凭证使用、presigned URL 有效期与权限 | R1/R9 |
| `cmd/server/main.go` | 启动装配、日志输出（审计面） | R6 |

每条发现记录：`证据（文件:行号 + 代码摘录）→ 影响分析 → CWE/OWASP ASVS 映射 → 修复建议`。

### 4.2 威胁建模（STRIDE × 资产）

| 资产 | S 仿冒 | T 篡改 | R 否认 | I 泄露 | D 拒绝服务 | E 提权 |
|------|--------|--------|--------|--------|------------|--------|
| 密文条目（OSS） | — | GCM 认证标签可检测 | — | MASTER_KEY 泄露即全量解密 | — | — |
| 元数据/审计日志 | — | 无完整性保护（审计面） | 审计不足（R6） | 元数据明文（R8） | — | — |
| JWT 会话 | 默认密钥可仿冒（R2 已修） | — | — | 7 天无吊销（R4） | — | 账号=全量数据（R3） |
| MASTER_KEY | — | — | — | 三处明文可见面（R1） | — | — |

### 4.3 基础设施与流程审计

- **tfstate**：远程 state 桶私有 ACL + SSE 是否落实（R1 缓解项）；
- **OSS 桶**：ACL、SSE-OSS、生命周期（30 天版本保留，R9）；
- **FC**：环境变量可见面、RAM 权限最小化、日志暴露面（R6）；
- **CI 供应链**：workflow 权限（GITHUB_TOKEN 最小化）、第三方 action 锁定版本、org secret 注入链（R1）；
- **运维流程**：备份导出权限与加密（R7）、密钥轮换制度（R1）、master-key.md 恢复路径核对。

### 4.4 审计报告模板（交付物）

```
1. 执行摘要（范围、方法、总体结论、关键发现 Top 5）
2. 审计范围与方法（对象 commit、工具链、时间）
3. 发现明细（按严重级排序；每条含证据、影响、CWE/ASVS、修复建议）
4. 风险登记册更新（R1–R11 状态：已验证 ✓ / 已缓解 ✓ / 待整改 ✗ / 新增 N）
5. 整改建议清单（P0 立即 / P1 短期 / P2 中期，对应安全云路线图输入）
6. 残余风险与结论
```

## 5. 里程碑

| 阶段 | 内容 | 产出 |
|------|------|------|
| M1 准备 | 检出 qtcloud-secret @ `cec7701`；安装工具链；确认无真实凭证依赖 | 基线清单、环境就绪 |
| M2 扫描 | 按 §3.1 矩阵全量扫描，原始结果归档 | 各层扫描原始报告 |
| M3 审计 | 代码审计（§4.1）+ 威胁建模（§4.2）+ 基础设施审计（§4.3） | 审计报告 v1 |
| M4 台账 | 漏洞登记、评级、排序、状态机初始化 | 漏洞台账 v1 |
| M5 复盘 | 对照 security.md R1–R11 汇总差距与整改队列；报告模板定稿 | 复盘报告、模板 v1 |

## 6. 工具与环境依赖

```bash
# Go 工具（govulncheck / gosec / osv-scanner / gitleaks）
go install golang.org/x/vuln/cmd/govulncheck@latest
go install github.com/securego/gosec/v2/cmd/gosec@latest
go install github.com/google/osv-scanner/cmd/osv-scanner@latest
go install github.com/gitleaks/gitleaks/v8@latest
# 通用（semgrep / trivy / tfsec）
pip install semgrep            # 或 brew install semgrep
brew install trivy tfsec       # 或按官方文档
# Flutter 侧（需 flutter SDK）
dart pub outdated             # src/studio 内执行
```

依赖：网络访问（GitHub / pub.dev / OSV API / 漏洞库）、Go ≥ 1.21、Flutter SDK（仅 studio 扫描需要）。

## 7. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 工具误报/漏报 | 人工复核机制：扫描结果须经代码级确认后才进台账；漏报以 R1–R11 对照兜底 |
| 对象基线漂移 | 锁定 commit `cec7701`；如需跟进新 commit 另行立项 |
| 真实凭证暴露 | **MASTER_KEY / 阿里云 AK 绝不进入实验室产物**；扫描仅针对代码与 IaC 静态面；gitleaks 扫描结果中的疑似凭据一律脱敏归档 |
| 结果过时 | 审计报告标注对象 commit 与审计日期；安全云产品化时以新 commit 重跑 |

## 8. 产出文件布局（规划）

```
examples/default/
├── README.md                        # 实验室入口（本 PoC 索引）
└── docs/
    └── poc-vuln-audit-qtcloud-secret.md   # 本规划
# 执行后新增：
#   scans/      # 各层原始扫描报告（M2）
#   findings/   # 漏洞台账 v1、审计报告 v1、复盘报告（M3–M5）
```
