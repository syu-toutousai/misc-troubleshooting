# FlClash 代理下 Stremio Torrentio 插件失效

## 症状

- 从 Clash Verge Rev 切换到 FlClash 后，Stremio 的 Torrentio 插件无法加载片源。
- 访问 `https://torrentio.strem.fun/manifest.json` 时 TLS 握手立即失败：

  ```
  curl: (35) TLS connect error: error:0A000126:SSL routines::unexpected eof while reading
  ```

- 同一台机器访问其他 HTTPS 站点（如 google.com）正常。
- 代理本身可用，只是 Torrentio 这个域名不行。

## 环境

- Linux
- FlClash（Mihomo 内核，TUN 模式 + fake-ip）
- 上游代理：AnyTLS 节点（香港出口）
- 客户端：Stremio
- 目标：`torrentio.strem.fun`（Cloudflare 前置）

## 诊断

### 1. 确认 Torrentio 本身是否存活

先绕过域名，直接指定 Cloudflare 的真实 IP。用 DoH 获取真实 IP，避免系统 DNS 被代理接管后误判：

```bash
curl -x http://127.0.0.1:7890 -H 'accept: application/dns-json' \
  'https://1.1.1.1/dns-query?name=torrentio.strem.fun&type=A'
```

返回：

```json
{"Answer":[{"data":"104.21.10.254"},{"data":"172.67.164.220"}]}
```

再用 `--connect-to` 强制连接真实 IP，但保持 SNI / Host 为域名：

```bash
curl -x http://127.0.0.1:7890 \
  --connect-to torrentio.strem.fun:443:104.21.10.254:443 \
  https://torrentio.strem.fun/manifest.json
```

结果：**HTTP 200**，成功拿到 manifest（Torrentio v0.0.15）。
说明 Torrentio 正常，问题在本地到 Torrentio 的路径上。

### 2. 比较「域名连接」与「IP 连接」

```bash
# 域名（失败）
curl -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' \
  https://torrentio.strem.fun/manifest.json

# IP（成功）
curl -x http://127.0.0.1:7890 \
  --connect-to torrentio.strem.fun:443:104.21.10.254:443 \
  -o /dev/null -w '%{http_code}\n' \
  https://torrentio.strem.fun/manifest.json
```

同时用 `openssl s_client` 观察握手：

```bash
echo | openssl s_client -proxy 127.0.0.1:7890 \
  -connect 104.21.10.254:443 \
  -servername torrentio.strem.fun
```

IP 连接可以正常完成 TLS，并拿到 `CN=strem.fun` 的证书。

### 3. 排除本地 DNS 污染

```bash
curl -x http://127.0.0.1:7890 'https://223.5.5.5/resolve?name=torrentio.strem.fun&type=A'
curl -x http://127.0.0.1:7890 -H 'accept: application/dns-json' \
  'https://1.1.1.1/dns-query?name=torrentio.strem.fun&type=A'
```

AliDNS 与 Cloudflare DoH 都返回正确的 Cloudflare IP（104.21.10.254 / 172.67.164.220）。
因此不是本地 DNS 污染，也不是 `GEOIP,CN,DIRECT` 误判。

## 根因

FlClash / Mihomo 把**域名**（而不是解析后的 IP）交给上游代理。
上游代理对 `torrentio.strem.fun` 这个目标域名存在阻断或无法解析，表现为：

```
CONNECT 隧道建立成功 → 客户端发出 TLS ClientHello → 立即被断开
```

用 IP 直连时，上游不检查域名，连接正常。
这是**上游代理的域名级阻断/解析问题**，不是 Torrentio 故障，也不是 FlClash 的 DNS 污染。

## 解决方案

### 方案 A：/etc/hosts 固定真实 IP（本次采用）

让系统直接把 `torrentio.strem.fun` 解析为 Cloudflare 真实 IP。
FlClash 的 DNS 配置为 `use-system-hosts: true`，会读取系统 hosts。
Stremio 连接到真实 IP 后，FlClash 以 IP 形式转交给上游代理，从而绕过域名阻断。

```bash
echo '104.21.10.254 torrentio.strem.fun' | sudo tee -a /etc/hosts
```

然后重启 FlClash（或清除其 DNS 缓存），使 hosts 生效。

验证：

```bash
curl -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' \
  https://torrentio.strem.fun/manifest.json
```

返回 `200` 即成功。

注意：

- 需要 root 权限。
- Cloudflare 的 Anycast IP 可能变化；若日后再次失效，重新用 DoH 查询并更新 hosts。
- 只影响 `torrentio.strem.fun`，不会影响其他站点。

复原：

```bash
sudo sed -i '/torrentio\.strem\.fun/d' /etc/hosts
```

### 方案 B：FlClash fake-ip-filter（备选）

在 FlClash 的 DNS 设置里，把 `torrentio.strem.fun` 加入 `fake-ip-filter`，
让该域名返回真实 IP 而不是 fake-ip：

1. 打开 FlClash → 配置 → 当前配置文件 → 编辑。
2. 在 `dns` 下加入：

```yaml
dns:
  fake-ip-filter:
    - torrentio.strem.fun
```

3. 保存并重新应用配置。

复原：删除该行并重新应用。

方案 B 不依赖固定 IP，能随 DNS 变化自动更新，但需要修改 FlClash 配置；
方案 A 不改 FlClash，但 IP 变化时需手动更新。

## 验证清单

- `curl -x http://127.0.0.1:7890 https://torrentio.strem.fun/manifest.json` 返回 200。
- Stremio 中 Torrentio 能正常列出片源。
- 其他站点访问不受影响。

## 经验

- 「代理能用」≠「所有域名都能用」。上游代理可能按目标域名做选择性阻断。
- 排查 TLS 立即 EOF 时，先比较「域名连接」与「IP + 正确 SNI」两种方式。
- 用 DoH 获取真实 IP，可以避免本地 DNS 被代理接管后误判。
- DNS 污染、GEOIP 误判、上游域名阻断是三类不同问题，需要分别验证。
