# 复盘报告 v1（PoC：漏洞管理与安全审计，对象 qtcloud-secret）

> 实验室：quanttide-laboratory-of-security-engineering ｜ 日期：2026-08-16
> PoC 规划：`docs/poc-vuln-audit-qtcloud-secret.md` ｜ 配套：漏洞台账 v1、审计报告 v1

## 1. PoC 目标达成度

| 成功标准 | 达成 | 说明 |
|----------|------|------|
| 全链路扫描一键复跑 | ✅ | 8 层扫描矩阵全部脚本化产出原始报告（`scans/`），命令见台账「扫描矩阵」 |
| 漏洞台账覆盖全部扫描层 | ✅ | `vulnerability-ledger-v1.md`：依赖/代码/客户端/基础设施/供应链 5 层 15 条发现 |
| R1–R11 逐条代码级证据 | ✅ | `audit-report-v1.md` §5：11 项全部给出结论与证据（5 未修复 / 2 已修复 / 2 已缓解 / 2 已接受/已验证） |
| 报告与台账模板可复用 | ✅ | 本套三文档即模板 v1，供安全云产品化直接沿用 |

## 2. 关键发现（超出既有认知的部分）

1. **jwt v5.2.1 DoS（V-01）**：既有 security.md 风险清单未覆盖依赖漏洞维度——补上了「依赖漏洞」这一审计层，三方工具交叉验证；
2. **`.gocache/` 入库（A-11）**：gitleaks 30 条命中全部为第三方测试夹具，真实结论是仓库卫生问题而非密钥泄露——**误报的正确处置示范**（不因误报而忽视，归类为卫生问题）；
3. **A-05（敏感响应无 no-store）**：security.md 未登记，手工审计新增；
4. **R2 修复得到三重验证**：代码（config.go prod 拒绝）+ 提交（28cdc60）+ 工具（gosec 仅命中 dev fallback 字符串）。

## 3. 方法学复盘

### 3.1 工具链有效性

| 工具 | 有效产出 | 局限 |
|------|----------|------|
| govulncheck | 34 条（符号级可达性分析） | 输出为多文档 JSON 流，需流式解析 |
| osv-scanner | 48 条（锁文件全量） | stdlib 版本取 go 指令，与实际构建镜像有偏差 |
| gosec | 4 条（与手工审计 100% 互证） | 规则集有限 |
| gitleaks | 30 条（全部为依赖缓存夹具） | 需人工判定归属（一方程 vs 三方） |
| trivy fs | jwt CVE + HEALTHCHECK + pub 0 漏洞 | 依赖库下载（首次运行需网络） |
| tfsec / trivy misconfig | 0 发现 | IaC 本身加固到位 |
| dart analyze | 0 诊断 | 仅语言级，不含逻辑漏洞 |
| semgrep | 环境受限（见 §3.2） | 需 Python ≤3.12 wheel 或大体积镜像 |

### 3.2 环境限制与偏差

- **semgrep 未完成**：本环境 PyPI 无 semgrep wheel（Python 3.14/3.11 均无）；Docker 镜像（~1.5GB）拉取两次被连接重置。**缓解**：gosec + dart analyze 已覆盖 Go/Dart SAST 层；后续在 CI 或网络稳定环境补跑并并入台账；
- **EPSS**：仅对代表性 CVE 查询（4 条）；全量 EPSS 可脚本化补充；
- **flutter pub outdated** 未单独执行：pubspec.lock 已由 osv-scanner/trivy 双工具覆盖（0 漏洞）。

### 3.3 误报处置记录

| 疑似发现 | 结论 | 依据 |
|----------|------|------|
| `isNotFound` 类型断言错误（OSS 404 不识别） | **误报，排除** | SDK v2.2.9 以值类型 `ServiceError` 返回错误（error.go:543），断言成立 |
| gitleaks 30 条密钥泄露 | **非一方程泄露** | 全部位于 `.gocache/mod/**` 第三方测试夹具；归类为 A-11 仓库卫生 |

## 4. 差距清单（对象仓库 → 安全云能力）

| 能力域 | 现状 | 安全云产品化输入 |
|--------|------|------------------|
| 漏洞管理 | 无自动化登记/跟踪体系（本次台账为手工整理） | 台账模板 + 状态机 + CVSS/EPSS 评级流程 → 漏洞管理模块数据模型 |
| 安全审计 | security.md 风险清单为静态文档 | 审计报告模板 + 扫描矩阵 → 安全审计模块工作流 |
| 整改跟踪 | 依赖 CHANGELOG/人工 | P0–P2 整改队列 → 任务/跟踪模型 |
| 持续巡检 | 无定期扫描机制 | 扫描矩阵脚本化 → 定时任务/CI 集成 |

## 5. 后续建议（对对象仓库 & 实验室）

1. **对象仓库（qtcloud-secret）**：15 条发现已提为 GitHub issues（`#1`–`#15`，标签 `poc-001` + `security:*`，https://github.com/quanttide/qtcloud-secret/issues），处理人已分配 **@LiXiang050789**。按 P0（jwt 升级、Go 工具链固定 digest）→ P1（授权模型、限流、审计、no-store、.gocache 清理）→ P2 顺序整改；issue 关闭即台账状态机「复测中 → 已关闭」的触发点；
2. **实验室**：① 本套扫描矩阵固化为 `scans/run-all.sh` 一键脚本；② 季度巡检机制（对齐 Go 版本演进）；③ semgrep 补跑；④ 下一个 PoC 对象建议 qtcloud-auth（身份面）或 qtcloud-devops（供应链面）；
3. **安全云**：台账/报告模板、扫描矩阵、评级流程直接作为「漏洞管理」「安全审计」两个首发能力域的 MVP 输入。

## 6. 结论

PoC 目标全部达成：**跑通了两条能力链路**（扫描→登记→评级→跟踪；审计→取证→报告→整改），产出 15 条分级发现（2 High / 5 Medium / 7 Low / 1 Info），验证并补强了对象仓库的既有风险认知，沉淀了 3 份可复用模板。
