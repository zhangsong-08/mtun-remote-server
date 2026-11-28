# MTUN 内网穿透 · 服务端

MTUN 的服务端程序：部署在公网服务器上，负责接受客户端接入、下发规则、监听公网端口并把流量转发到各客户端节点。

- 控制面 TLS 1.3（默认 `:10000`），管理密码与 PIN 只保存 PBKDF2-SHA256 哈希，接入密码可随时查看
- 本机 Web 控制台（默认 `127.0.0.1:10004`）：状态、节点、转发规则、加密、面板管理、版本更新
- 每节点独立接入密码；支持「无密码申请接入」，由管理员在面板批准或拒绝
- Linux / Windows 双平台，安装脚本自动注册开机自启

> 客户端请到 [mtun-remote-client](https://github.com/zhangsong-08/mtun-remote-client) 下载安装。

---

## 一、安装

### Linux

```bash
curl -fsSL https://raw.githubusercontent.com/zhangsong-08/mtun-remote-server/main/scripts/install.sh -o install.sh
sudo bash install.sh
```

脚本会逐项询问安装选项（安装目录、面板地址、控制面端口、默认接入密码、管理密码、版本、下载源），**直接回车使用方括号内的默认值**。随后自动完成下载、校验、生成配置、注册 systemd 服务 `mtun-server`。

```bash
sudo bash install.sh -y                                  # 全部默认，不询问
sudo bash install.sh -m https://ghfast.top/              # 国内服务器加速
sudo bash install.sh --panel 0.0.0.0:10004 -y            # 允许远程访问面板
sudo bash install.sh --file ./mtun-linux-amd64 -y        # 离线安装
sudo bash install.sh -u                                  # 卸载
```

### Windows

管理员 PowerShell：

```powershell
curl.exe -fsSL https://raw.githubusercontent.com/zhangsong-08/mtun-remote-server/main/scripts/install.ps1 -o install.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1
```

默认安装到 `C:\Program Files\mtun`，加 `-Yes` 跳过询问。参数：`-AdminPassword`、`-JoinPassword`、`-Panel`、`-Control`、`-InstallDir`、`-Version`、`-Mirror`、`-LocalFile`、`-Yes`、`-Uninstall`、`-Help`。

### 手动运行

```bash
mtun default server > server.yaml
mtun passwd -c server.yaml     # 面板管理密码（至少 8 位）
mtun access -c server.yaml     # 默认接入密码
mtun server -c server.yaml
```

打开 `http://127.0.0.1:10004` 进入控制台。

---

## 二、端口

| 端口 | 用途 |
| --- | --- |
| `10000/tcp` | 控制面（客户端接入），必须公网可达 |
| `10004/tcp` | 控制台，默认仅本机 |
| `10100-10999/tcp` | 客户端发布端口范围（`publish.port_range` 可改） |

云服务器安全组需要放行 `10000` 与用到的发布端口。

---

## 三、密码与准入策略

| 凭据 | 作用 | 设置方式 |
| --- | --- | --- |
| 管理密码 | 登录服务端控制台 | `mtun passwd -c server.yaml` |
| 敏感操作 PIN | 面板敏感操作二次验证（6 位） | `mtun pin -c server.yaml -new 123456` |
| 默认接入密码 | 未设置独立密码的节点使用 | `mtun access -c server.yaml` |
| 节点独立接入密码 | 指定节点专用 | 控制台「加密」页，或批准接入申请时自动生成 |
| 接入申请 | 无密码客户端申请接入 | 「加密」页批准/拒绝，可整体开关 |

管理密码与 PIN 只保存哈希。接入密码（默认密码、节点独立密码）可在「加密」页**随时查看，次数不限**；
服务端把可读密码保存在 `server.yaml.nodepass.json`（`0600`，仅本机可读）。

### 无密码申请接入

1. 服务端「加密」页保持「允许无密码申请接入」开启（默认开启）
2. 客户端只填服务端地址、不填密码启动
3. 「加密 → 接入申请」出现该节点，显示节点名、来源 IP、版本、申请时间
4. 点「同意」：系统立刻生成该节点的独立接入密码并自动下发给客户端，客户端无需人工干预即可接入；密码在「加密」页可反复查看
5. 点「拒绝」：客户端收到原因，30 秒后重试

申请只在内存中保留 10 分钟；同一 IP 最多 3 条待处理。

### 同密码多设备与换设备同步

- 多台设备使用同一接入密码时，服务端**不会断开**任何客户端，只在「加密 → 密码复用提醒」记录一条（含次数与时间），可随时清除。
- 新设备注册时若未上报转发规则，服务端会把密码所属节点的规则自动同步给新设备（`rules_sync`）。
- 因此「换个设备、填同一个密码」即可拿回原有转发配置。

---

## 四、多设备管理

每台设备装客户端后成为独立节点，服务端负责统一视图：

| 操作 | 位置 |
| --- | --- |
| 查看在线/离线节点、直连/中继、覆盖 IP | 控制台「节点」 |
| 踢下线、临时或永久封禁、按时间段/网段/系统限制 | 「节点 → 访问策略」 |
| 查看或修改任意节点的转发规则 | 「转发规则」 |
| 给节点设/重置独立密码，关闭默认密码 | 「加密」 |
| 改端口、管理密码、开机自启 | 「面板管理」 |
| 在线升级并自动重启 | 「版本更新」 |

命令行（本机执行，自动读取 `.apitoken`）：

```bash
mtun status -c server.yaml      # 概况
mtun nodes  -c server.yaml      # 节点列表
mtun watch  -c server.yaml      # 实时刷新
mtun kick   -c server.yaml -id 节点ID [-ban 60s]
```

---

## 五、远程访问控制台

```bash
ssh -L 10004:127.0.0.1:10004 root@服务器IP
# 本机打开 http://127.0.0.1:10004
```

也可以用安装脚本 `--panel 0.0.0.0:10004` 直接暴露，但务必配合防火墙与强密码。

---

## 六、升级与卸载

```bash
mtun update -c /opt/mtun/configs/server.yaml --check
sudo mtun update -c /opt/mtun/configs/server.yaml
```

或使用控制台「版本更新」一键升级。升级只替换程序，保留配置、密码与规则。

```bash
sudo bash install.sh -u          # Linux 卸载
powershell -File .\install.ps1 -Uninstall   # Windows 卸载
```

---

## 七、常见问题

**客户端一直连不上**
服务端必须是 v0.5.0 及以上；确认 `10000/tcp` 放行；确认客户端使用的密码存在于「加密」页，或申请已被批准；确认没有关闭「允许默认密码接入」却仍在用默认密码。

**忘记默认接入密码**
`mtun access -c server.yaml` 重设后重启服务端，所有使用默认密码的客户端都要同步更新。

**忘记面板密码 / PIN**
`mtun passwd -c server.yaml`、`mtun pin -c server.yaml -new 123456`。

**申请列表为空但客户端说在申请**
检查「允许无密码申请接入」是否被关闭；申请只保留 10 分钟；服务端日志会记录 `join request queued`。

**端口发布后访问不通**
确认规则在客户端侧存在且运行中、端口在 `publish.port_range` 内、防火墙/安全组已放行。

---

## 八、从源码构建

需要 Go 1.26+：

```bash
git clone https://github.com/zhangsong-08/mtun-remote-server.git
cd mtun-remote-server
CGO_ENABLED=0 go build -trimpath -ldflags "-s -w \
  -X mtun/internal/version.Version=v0.6.0 \
  -X mtun/internal/version.Repo=zhangsong-08/mtun-remote-server" \
  -o mtun ./cmd/mtun
```

---

## License

MIT © 2026 song08
