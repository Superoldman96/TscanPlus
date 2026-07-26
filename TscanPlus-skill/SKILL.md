---
name: tscanplus
description: >-
  Operates TscanPlus security scanner via MCP tools or CLI for authorized targets only.
  Use when the user mentions TscanPlus, port/URL/POC/subdomain scanning, MCP integration,
  ip_scan, tscan_scan, ak_verify, cloud AK leak credential check, or recon on IPs, domains,
  or URLs in any AI assistant with MCP support.
---

# TscanPlus 扫描助手

通过 **MCP 工具**（首选）或 **CLI** 驱动 TscanPlus（与无影 GUI 版共享 `config.yaml`、`config.db`）。参数语义对齐 **TscanClient** 命令行版（`-m` 八大模块、`-h/-u/-d/-ck` 等）。

> 本文档为 Agent Skill，可放入宿主技能目录，或复制章节到自定义系统提示。对话示例见 [examples.md](examples.md)。

## 授权（强制）

- 仅对用户**明确拥有书面授权**的目标扫描（自有 lab、渗透项目 scope 内）。
- 未获授权时：**拒绝扫描**，并说明原因。
- 默认避免：全端口 `1-65535`、大范围 C 段、生产环境、开启 `poc_check`/`pwd_check`/`poc_full`，除非用户明确要求。
- 资产过多时不要一次开启全部模块，以免耗时过久或占满 CPU。
- 扫描前用一句话复述：目标、模块、是否含 POC/爆破。

## 产品能力概览

TscanPlus / CLI 集成八大安全检测模块：

| 模块 | CLI `-m` | MCP 单工具 | 说明 |
|------|----------|------------|------|
| 端口扫描 | `port` | `ip_scan` | IP/CIDR、存活探测、端口、可选服务指纹/POC/弱口令 |
| Web 指纹 | `url` | `url_scan` | URL 指纹识别、Title、可选联动 POC |
| POC 验证 | `poc` | `poc_scan` | xray 1.0 格式 POC，可按指纹或全量 |
| 弱口令 | `crack` | `pwd_crack` | 多协议爆破，`targets` 为 `host:port` |
| 目录枚举 | `dir` | `dir_scan` | 路径爆破，可自定义字典 |
| JS 敏感信息 | `js` | `js_scan` | JS 文件中密钥、接口等 |
| 子域名 | `domain` | `subdomain_scan` | 字典 + 可选 API（key 在 config.yaml） |
| 空间测绘 | `cyber` | `cyber_search` | Hunter/FOFA 等（引擎在 config.yaml） |
| AK 泄露验证 | —（GUI AK管理） | `ak_verify` | OSS/云主机/微信/钉钉/飞书/地图/海康萤石/听云/百度人脸/绿盟 凭据 API 验证 |

**多模块联动**使用 `tscan_scan`，`modules` 对应 `-m`，流程与 GUI 项目管理一致：

**cyber → domain → port → crack → url → poc → dir → js**

前一项的**全部结果**会作为下一项的输入。常用组合（授权 lab 内）：

| 场景 | `modules` / CLI `-m` |
|------|----------------------|
| 内网主机摸底 | `port,url,poc` 或 `port,poc,crack` |
| Web 专项 | `url,poc,dir,js` |
| 域名资产 | `domain,port,url,poc` |
| 测绘后深挖 | `cyber,port,url,poc` |

## MCP 工具选型

| 用户意图 | 优先工具 | 说明 |
|----------|----------|------|
| 单 IP/CIDR 看端口 | `ip_scan` | `target` 必填；先小范围 `ports` |
| 多目标只扫端口+弱口令+POC | `ip_scan` 或 `tscan_scan` | `ip_scan` 用 `pwd_check`/`poc_check` |
| 一批 URL 指纹/Web | `url_scan` | `targets` 逗号分隔 |
| 已知 URL 打 POC | `poc_scan` | 慎用 `poc_full` |
| 弱口令 | `pwd_crack` | `targets`: `192.168.1.1:22,192.168.1.1:3306` |
| 目录 / JS | `dir_scan` / `js_scan` | 需完整 URL |
| 子域名 | `subdomain_scan` | `domains`；API 需 config |
| 空间测绘 | `cyber_search` | `query` 对齐 `-ck` |
| 多阶段、结果传递 | `tscan_scan` | 大任务可分阶段执行 |
| 云 AK 泄露验证 | `ak_verify` | `kind=oss/server/wechat/dingtalk/feishu/map/hikys/tingyun/baiduface/nsfocus` |

## 通用参数（MCP / CLI）

| MCP / 含义 | CLI | 默认 | 说明 |
|------------|-----|------|------|
| `project` | `-pr` | 未指定→`MCP` | 写入 `config.db` 的项目名；见下文「项目 MCP」 |
| `fresh_project` | — | `false` | 指定 `project` 时是否先清空该项目 |
| `proxy` | `-proxy` | 配置全局代理 | HTTP/SOCKS5；**端口扫描仅 SOCKS5** |
| `include_results` | — | `true` | 响应 JSON 是否带 `data.results` |
| `result_limit` | — | `200` | 每类结果最多条数，最大 `2000` |
| `ping_scan` | 未用 `-np` 即探测 | `true` | 存活探测 |
| `smart_scan` | 未用 `-nosmart` | `true` | 大网段启发式扫描（`tscan_scan`） |
| `timeout` | `-time` | `3` | 通用超时（秒） |
| `web_timeout` | `-wt` | `10` | Web 超时（秒） |

**结果存放：**

- MCP 工具返回 JSON（`data.results`、`data.counts`）。
- CLI 写日志 `TscanPlus-Result.txt` 及各模块 txt，并写入 **`config.db`**（可与 GUI 共用）。

## 项目名 `MCP` 的行为

| 情况 | 行为 |
|------|------|
| 未传 `project` | 使用 **`MCP`**，**本次工具调用前**自动清空该项目数据 |
| 传 `project=自定义名` | 默认**追加**；清空则 `fresh_project=true` |

**GUI 注意：** 单工具（如 `ip_scan`）可能不在项目列表显示行，但 `ipscan` 等表可有 `Project='MCP'`；`tscan_scan` 会在 `project` 表登记。

---

## 各模块参数（MCP ↔ CLI）

### 1. 端口扫描 `ip_scan`（`port`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `target` | `-h` | 必填 | `192.168.1.1`、`192.168.1.0/24`、范围 |
| `ports` | `-p` | `Top100` | `22`、`1-65535`、`22,80,443` |
| `thread` | `-t` | `600` | 并发 |
| `timeout` | `-time` | `3` | 秒 |
| `ping_scan` | 默认探测 / `-np` 关闭 | `true` | 存活探测 |
| `ip_finger` | 服务指纹 | 配置项 | 服务识别 |
| `poc_check` | 联动 POC | `false` | 开放端口转 URL 后 POC |
| `pwd_check` | 联动 crack | `false` | 弱口令 |
| `proxy` | `-proxy` | — | 建议 SOCKS5 |

`tscan_scan` 额外：`ports_add`→`-pa`，`exclude_hosts`→`-hn`，`smart_scan`→`-nosmart` 取反。

### 2. Web 指纹 `url_scan`（`url`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `targets` | `-u` / `-uf` 内容 | 必填 | URL，逗号或换行分隔 |
| `thread` | URL 线程 | `50` | |
| `web_timeout` | `-wt` | `10` | 秒 |
| `finger` | `-finger` | `tiny` | `tiny` / `min` / `all` |
| `cookie` | `-cookie` | — | 认证场景 |
| `poc_check` | 联动 POC | `false` | |
| `proxy` | `-proxy` | — | |

### 3. POC `poc_scan`（`poc`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `targets` | `-u` | 必填 | HTTP(S) URL |
| `thread` | `-num` | `20` | POC 并发 |
| `poc_full` | `-full` | `false` | `true` 时不匹配指纹，扫全部 POC |
| `poc_name` | `-pocname` | — | 如 `weblogic` |
| `poc_level` | `-poclevel` | `1+2+3+4+5` | 级别过滤 |
| `proxy` | `-proxy` | — | |

POC 路径在 `config.yaml`（xray 1.0 格式），与 GUI 一致。

### 4. 弱口令 `pwd_crack`（`crack`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `targets` | `-h` | 必填 | **`host:port`**，逗号分隔 |
| `services` | `-s` | `all` | `ssh,mysql,rdp` 等 |
| `user` | `-user` | 内置字典 | 逗号分隔多个 |
| `pwd` | `-pwd` | 内置字典 | |
| `cmd` | `-c` | `whoami` | 成功后执行命令 |
| `thread` | `-br` | `1` | 爆破线程 |
| `timeout` | `-time` | `3` | |

### 5. 子域名 `subdomain_scan`（`domain`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `domains` | `-d` | 必填 | 逗号分隔主域 |
| `sub_api` | `-api` | `false` | API key 在 `config.yaml` |
| `sub_dict` | `-dc` | 内置 | **绝对路径** |
| `ports` | 联动扫描端口 | `80,443` | 发现子域后的端口 |
| `proxy` | `-proxy` | — | |

### 6. 目录 `dir_scan`（`dir`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `urls` | `-u` | 必填 | 基 URL |
| `thread` | `-ds` | `20` | |
| `dict` | `-dd` | 内置 10k | **绝对路径** |
| `timeout` | `-time` | `3` | |
| `proxy` | `-proxy` | — | |

### 7. JS `js_scan`（`js`）

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `urls` | `-u` | 必填 | |
| `timeout` | `-time` / Web | `3` | |
| `proxy` | `-proxy` | — | |

### 8. 空间测绘 `cyber_search`（`cyber`）

与 GUI「空间测绘」页字段下拉一致；`field` 决定如何把 `query` 转成各引擎 API 语句。

| MCP 参数 | CLI | 默认 | 说明 |
|----------|-----|------|------|
| `query` | `-ck` | 必填 | 见下表「query 写法」 |
| `field` | 查询类型 | `domain` | 见下表 |
| `engines` | — | config 已启用 | 逗号分隔引擎名 |
| `project` / `fresh_project` | `-pr` | `MCP` / `false` | 同通用参数 |
| `include_results` / `result_limit` | — | `true` / `200` | 同通用参数 |

**`field` 可选值（对齐 GUI）：**

| `field` | GUI 名称 | `query` 示例（不加引号） |
|---------|----------|-------------------------|
| `domain` | 域名 | `example.com` |
| `ip` | IP 地址 | `1.1.1.1`、`192.168.1.0/24` |
| `port` | 端口 | `80`、`3306` |
| `product` | 应用 | `nginx`、`apache` |
| `title` | 标题 | `管理后台` |
| `service` | 服务 | `mysql`、`ssh` |
| `cert` | 证书 | 证书关键词 |
| `icp` | 备案 | 备案号或主体名 |
| `body` | Body | 页面正文关键词 |
| `icon` | Icon | Icon URL 或 hash |
| `custom` | 自定义 | 各平台完整语法，如 `domain="example.com" && port="443"` |

**用法要点：**

- **推荐**：选具体 `field`，`query` 只填关键词（与 GUI 搜索框相同），不要写 `domain="xxx"` 这类包装语法。
- **`custom`**：各测绘平台语法不同，一般无法跨平台通用；需自行按平台调试。
- **多关键词**：逗号分隔，如 `example.com,example.net`（与 GUI 一致）。
- CLI `-ck` 常写完整语句时，MCP 应设 `field=custom`；或 `-ck example.com` 配合默认 `domain`。

`tscan_scan` 中用 `cyber_query` 传查询内容，`cyber_field` 传上表字段类型（默认 `domain`）。

### 9. AK 泄露 API 验证 `ak_verify`（GUI「AK管理」）

对齐 GUI **AK管理** 全部 Tab：用官方 API 校验泄露凭据是否仍可用。

**`kind` 一览：**

| `kind` | GUI Tab | 验证方式 |
|--------|---------|----------|
| `oss` | OSS存储管理 | 列存储桶 |
| `server` | 云主机管理 | 列举实例 |
| `wechat` | 微信利用 | 获取 AccessToken |
| `dingtalk` | 钉钉利用 | 获取 AccessToken |
| `feishu` | 飞书利用 | 获取 tenant_access_token |
| `map` | 地图相关 | 调用地图 API（默认地理编码） |
| `hikys` | 海康萤石 | 云眸/iSecure/萤石验活 |
| `tingyun` | 听云 | 获取/探测 AccessToken |
| `baiduface` | 百度人脸 | 获取 AccessToken（可选人脸检测） |
| `nsfocus` | 绿盟巡检 | UTS 登录 Token / RSAS Basic 编码 |

**通用参数：**

| MCP 参数 | 默认 | 说明 |
|----------|------|------|
| `kind` | 必填 | 见上表 |
| `access_key` | 多数必填 | 各模块主键（见下「字段映射」） |
| `secret_key` | 多数必填 | 各模块密钥 |
| `fake_ip` | — | 伪造客户端 IP（微信/钉钉/飞书/地图/海康/听云/百度/绿盟） |
| `include_token` | `true` | 是否返回 access_token（敏感） |
| `body_limit` | `2000` | 响应 body 截断，最大 `8000` |

**字段映射（`access_key` / `secret_key`）：**

| `kind` | `access_key` | `secret_key` | 其他关键参数 |
|--------|--------------|--------------|--------------|
| `oss` | AccessKey | SecretKey | `provider` 必填；`session_token`/`ext_json`/`include_details` |
| `server` | AccessKey | SecretKey | `provider` 必填；`session_token`/`include_details` |
| `wechat` | AppID / CorpID | AppSecret / CorpSecret | `mode`: `oa`（默认）/`mini`/`work` |
| `dingtalk` | AppKey | AppSecret | `mode`: `standard`（默认）/`exclusive`；专属需 `domain` |
| `feishu` | App ID | App Secret | 可选 `base_url` |
| `map` | API Key | — | `service`（默认 `amap_geocode`）；`origin`/`destination` |
| `hikys` | ClientID / AppKey | ClientSecret / AppSecret | `provider`: `hikcloud`/`isecure`/`ys7`；isecure 需 `host`；可选 `action`/`fetch_token` |
| `tingyun` | API Key | Secret Key | `host` 必填；或只传 `access_token` 探测 |
| `baiduface` | API Key | Secret Key | 可选 `image`（Base64，继续人脸检测） |
| `nsfocus` | 用户名 | 密码 | `mode`: `uts`（默认，需 `base_url`）/`scanner`（RSAS Basic） |

**OSS / 云主机 `provider`：**

| `kind` | `provider` |
|--------|------------|
| `oss` | `aliyun` `tencent` `huawei` `tianyi` `jdcloud` `ksyun` `aws` `minio` `baidu` `qiniu` `yidong` `liantong` `qingcloud` `upyun` |
| `server` | `aliyun` `tencent` `jdcloud` `huawei` `aws` `ucloud` `baidu` `volcengine` |

**地图 `service`：** `amap_walking` `amap_geocode` `amap_mp_regeocode` `tianditu_staticimage` `tianditu_search` `tianditu_wmts` `baidu_web_search` `baidu_ios_search`

**响应要点（`data`）：**

- `valid`：凭据是否可用
- 多数模块：`token`（可关）、`body`、`error`
- `oss`：`bucket_count`、可选 `buckets`
- `server`：`region_count`、`instance_count`、可选 `instances`
- 凭据无效时仍返回 JSON（`success: true`，`valid: false`）；参数错误才是工具级错误

**授权注意：** 仅验证用户明确授权范围内的凭据；不要在对话中完整回显 SecretKey / AppSecret。

### 10. 综合扫描 `tscan_scan`

在单模块参数基础上，常用额外字段：

| MCP 参数 | CLI | 说明 |
|----------|-----|------|
| `targets` | `-h/-u/-d/-ck` 等 | 按 `modules` 解析目标 |
| `modules` | `-m` | 默认 `port,url,poc` |
| `domains` | `-d` | `domain` 模块主域 |
| `cyber_query` | `-ck` | 测绘语句 |
| `ports` / `ports_add` | `-p` / `-pa` | |
| `exclude_hosts` | `-hn` | |
| `thread` / `url_thread` / `poc_thread` / `dir_thread` | `-t` 等 | |
| `finger` | `-finger` | |
| `poc_full` / `poc_name` / `poc_level` | POC 相关 | |
| `crack_*` | `-s/-user/-pwd/-c/-br` | 弱口令 |
| `sub_api` / `sub_dict` | 子域 | |
| `dir_dict` | `-dd` | |

---

## 推荐参数（默认保守）

**单主机端口（首选入门）：**

```yaml
tool: ip_scan
target: "10.0.0.1"
ports: "80,443,8080,8443,22"
ping_scan: true
ip_finger: false
poc_check: false
pwd_check: false
thread: 200
timeout: 3
include_results: true
result_limit: 100
```

**内网 C 段（lab，控制范围）：**

```yaml
tool: tscan_scan
targets: "192.168.1.0/24"
modules: "port,url"
ports: "Top100"
thread: "300"
ping_scan: true
smart_scan: true
include_results: true
result_limit: 200
```

**仅在用户明确要求时启用：** `poc_check`、`pwd_check`、`poc_full`、`modules` 含 `poc`/`crack`、全端口。

---

## MCP 接入

### stdio（推荐）

```json
{
  "mcpServers": {
    "tscanplus": {
      "command": "/绝对路径/TscanPlus",
      "args": ["mcp", "stdio"]
    }
  }
}
```

### Streamable HTTP（推荐远程传输）

```bash
TscanPlus mcp serve -listen 127.0.0.1:8088
# 或显式指定：-transport streamable
```

```json
{
  "mcpServers": {
    "tscanplus": {
      "url": "http://127.0.0.1:8088/mcp"
    }
  }
}
```

### HTTP+SSE（旧版兼容）

```bash
TscanPlus mcp serve -listen 127.0.0.1:8088 -transport sse
```

```json
{
  "mcpServers": {
    "tscanplus": {
      "url": "http://127.0.0.1:8088/sse"
    }
  }
}
```

GUI：**AI 辅助 → MCP 服务配置** 可选择传输模式（Streamable HTTP / HTTP+SSE），启停服务并导出 Skill 模板 zip。

各宿主引用见 [README.md](README.md)。

---

## 调用后如何汇报

解析 JSON（`success`、`message`、`data`）：

1. **摘要**：目标、模块、开放端口/URL 数、高危 `poc_vul`。
2. **表格化列表**：`results.ipscan` / `urlscan` / `poccheck` / `pwdcrack` / `dirscan` / `jsfinder` / `subdomain` / `cyber` 关键字段。
3. **`project_cleared: true`**：已清空默认 `MCP` 项目旧数据。
4. `counts` > `result_limit`：说明仅返回前 N 条，全量在 `config.db` 或 GUI。

不要只回复「扫描完成」。

---

## 常见工作流

1. **IP → Web → 漏洞：** `ip_scan`（常见 Web 端口）→ 拼 URL → `url_scan` → 用户确认 → `poc_scan`
2. **子域 → 端口 → Web：** `subdomain_scan` → `tscan_scan`（`modules=port,url`）→ 按需 POC
3. **测绘 → 扫描：** `cyber_search` → 提取 IP/URL → 用户确认 → `ip_scan` / `url_scan`
4. **单项目持续：** 全程 `project=pentest-xx`，阶段结束用 `fresh_project=true` 重扫
5. **AK 泄露验证：** 发现密钥 → `ak_verify`（按来源选 `kind`：云 AK 用 `oss`/`server`，微信钉钉飞书/地图/海康/听云/百度人脸/绿盟对应各自 kind）→ 汇报 `valid` 与摘要

## 排错

| 现象 | 处理 |
|------|------|
| 看不到 TscanPlus 工具 | 检查 MCP 配置、二进制绝对路径、重载 MCP |
| 长时间无响应 | 同步阻塞扫描；缩小 `ports`/`targets`，先关 POC/爆破 |
| 卡在 `xxx open` | 升级版本；重启 `mcp serve` 或 GUI 内 MCP 服务 |
| GUI 无 MCP 项目行 | 单工具可能不写 `project` 表；查库表或改用 `tscan_scan` |
| 测绘/子域无结果 | 检查 `config.yaml` 中 API Key、Engines |
| `ak_verify` 失败 / valid=false | 核对 `kind` 与字段映射；OSS/云主机要 `provider`；MinIO 要 `ext_json.endpoint`；钉钉专属要 `domain`；绿盟 UTS 要 `base_url`；海康 isecure 要 `host` |
| 结果与 GUI 不一致 | 共用同一 `config.yaml` / `config.db` |

## MCP 不可用时的 CLI

```bash
# 默认：port + url + poc（-h 触发）
TscanPlus -h 192.168.1.1/24

TscanPlus -m port -h 192.168.1.1 -p 80,443,3306 -t 600
TscanPlus -m url,poc,dir,js -uf urls.txt -finger tiny
TscanPlus -m domain,port,url,poc -d example.com -api
TscanPlus -m crack -h 192.168.1.1 -p 22 -s ssh -user root -pwd 123456
TscanPlus -m cyber,port,poc -ck 'domain="example.com"'
TscanPlus -pr MyProject -m port,url,poc -h 192.168.1.1

TscanPlus mcp stdio
TscanPlus mcp serve -listen 127.0.0.1:8088
TscanPlus mcp serve -listen 127.0.0.1:8088 -transport sse
```

CLI 不自动清空项目；MCP 未指定 `project` 时默认 `MCP` 且每次调用前清空。
