# PoC 规划：漏洞管理与安全审计（对象：qtcloud-auth）

> 实验室：`quanttide-laboratory-of-security-engineering`（量潮安全工程实验室）
> 状态：**已执行完成**（2026-08-16）。审计产物见 `../findings-auth/`（漏洞台账 v1、审计报告 v1、复盘报告 v1）
> 关联：量潮安全云首发能力域——漏洞管理、安全审计；PoC-001（qtcloud-secret）的同类复跑

## 1. 背景与目标

沿用 PoC-001 的方法与模板（[poc-vuln-audit-qtcloud-secret.md](./poc-vuln-audit-qtcloud-secret.md)），对 **qtcloud-auth（量潮身份云）** 执行同类审计。身份云是安全关键面：**密码处理、JWT 签发、OAuth2 令牌、SMS 验证码、管理员端点**均在此仓库。

目标：验证 PoC-001 沉淀的扫描矩阵与报告模板在**第二个对象**上的可复用性，产出身份云专项发现。

### 成功标准

1. 8 层扫描矩阵在 auth 仓库一键复跑（无 studio → dart analyze 层跳过，替代以 pyproject 检查）；
2. 漏洞台账覆盖全部扫描层，含 CVSS/EPSS 与状态；
3. 审计报告覆盖身份云重点面：密码哈希（Argon2id）、JWT 签发/验签（RS256 私钥）、OAuth2 授权码流程、SMS 验证码、ADMIN_TOKEN 保护、RDS 存储；
4. 报告与台账模板复用 PoC-001（差异仅对象与发现）。

## 2. 审计对象：qtcloud-auth @ `f017ffa`

### 2.1 架构速览

```
客户端 ──密码/SMS 登录──▶ FC 应用服务（Go provider）
                            ├── 密码：Argon2id 哈希（x/crypto）
                            ├── JWT：RS256 签发（JWT_PRIVATE_KEY，org secret 注入）
                            ├── OAuth2：go-oauth2 v4（授权码/密码模式）
                            ├── SMS：阿里云 dysmsapi 验证码
                            └── 存储：RDS PostgreSQL（gorm）
        ADMIN_TOKEN（org secret）──▶ /admin/** 运维端点
```

### 2.2 攻击面清单

| 层面 | 范围 | 关键文件 |
|------|------|----------|
| 代码面（服务端） | Go provider | `cmd/server/main.go`、`internal/api/{oauth,password,sms,sms_aliyun,admin,middleware,handler,response}.go`、`internal/app/app.go`、`internal/model/{user,role,verification_code}.go`、`internal/store/store.go` |
| 基础设施面 | Terraform IaC | `manifests/terraform/*.tf`（FC + **RDS** + state） |
| 供应链面 | CI/CD | `.github/workflows/deploy-provider.yml` |
| 依赖面 | Go | `go.mod`（jwt v5.3.0 / go-oauth2 v4.5.4 / gorm / x/crypto） |
| 测试面 | Python | `pyproject.toml`（集成测试，无锁文件） |

## 3. 扫描矩阵（PoC-001 同款）

| 层面 | 工具 | 目标 |
|------|------|------|
| Go 依赖漏洞 | govulncheck | src/provider |
| 全依赖匹配 | osv-scanner | go.mod |
| Go 静态分析 | gosec | src/provider |
| 密钥泄露 | gitleaks（全历史） | 仓库 |
| IaC | tfsec + trivy misconfig | manifests/terraform |
| 镜像/FS | trivy | 仓库全量 |
| 语义规则 | semgrep | 环境受限（同 PoC-001，跳过） |
| Dart SAST | dart analyze | 不适用（无 studio） |

## 4. 审计重点（身份云专项）

- **密码链路**：Argon2id 参数、哈希验证、错误信息（用户枚举）、重置流程
- **JWT 链路**：RS256 私钥管理、exp/iss/aud 校验、算法混淆、token 撤销
- **OAuth2 链路**：授权码流程、client 校验、redirect_uri 校验、token 存储
- **SMS 链路**：验证码生成/校验/过期、频率限制、重放
- **管理端点**：ADMIN_TOKEN 比较方式（常数时间）、鉴权覆盖
- **存储**：RDS 连接凭证、gorm 查询注入面、敏感字段落盘
- **基础设施**：RDS 公网/加密、FC 环境变量、state 桶

## 5. 里程碑与产物

| 阶段 | 内容 | 产出 |
|------|------|------|
| M1 准备 | 基线 `f017ffa`；工具链复用 PoC-001 | 基线确认 |
| M2 扫描 | 矩阵全量扫描 | `scans/` 原始报告 |
| M3 审计 | 身份云重点面逐文件取证 | 审计报告 v1 |
| M4 台账 | 漏洞登记/评级/状态机 | 漏洞台账 v1 |
| M5 复盘 | 方法复用性评估 + 差距清单 | 复盘报告 v1 |
