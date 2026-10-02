# 部署說明：Coolify + Cloudflare Tunnel + 容器 Port

## 結論

只靠三件事：

1. `compose.yaml` 把容器 port 發佈到 host 的 `127.0.0.1:PORT`。
2. Coolify 設定 `PORT`，且每個 app 的 `PORT` 都不一樣。
3. Cloudflare Tunnel 的 hostname 指向 `http://127.0.0.1:PORT`。

## 資料流

```
瀏覽器 → Cloudflare → cloudflared（在 Docker host 上）→ 127.0.0.1:PORT → 容器內 app（監聽 PORT）
```

Coolify 的 Traefik 不參與：Coolify 的 Domains 欄位留空，也不需要 `SERVICE_FQDN_*`。

## 關鍵設定

`compose.yaml`：

```yaml
services:
  app:                      # 服務名稱隨意
    build: .
    restart: unless-stopped
    environment:
      - PORT=${PORT:?set PORT}
    ports:
      - "127.0.0.1:${PORT}:${PORT}"
```

`Dockerfile`（nginx 版本，其他語言原則相同）：

```dockerfile
ENV PORT=8080
COPY nginx.conf.template /etc/nginx/templates/default.conf.template
HEALTHCHECK CMD wget -qO- http://127.0.0.1:${PORT}/ >/dev/null || exit 1
```

```nginx
# nginx.conf.template：官方 nginx 映像啟動時會用 envsubst 替換 ${PORT}
server { listen ${PORT}; ... }
```

Coolify：Environment Variables 設 `PORT=<唯一 port>`，Domains 留空。

Cloudflare Zero Trust：Tunnels → 該 tunnel → Public Hostname，Service 填 `HTTP` + `127.0.0.1:<port>`。

## 六個關鍵點

| # | 關鍵點 | 說明 |
|---|---|---|
| 1 | app 必須監聽 `$PORT`，且綁 `0.0.0.0` | 容器內綁 `127.0.0.1` 的話，host 的 port 轉發連不進去。nginx 預設已是全部網卡。Node/Vite 要加 `--host 0.0.0.0` 或設 `HOST=0.0.0.0` |
| 2 | `ports` 兩邊用同一個 `${PORT}` | 對外與容器內 port 相同，只需維護一個變數 |
| 3 | `127.0.0.1:` 前綴 | 只開 host 本機。省略的話會綁 `0.0.0.0`，外網能直連 port，繞過 Cloudflare |
| 4 | `${PORT:?set PORT}` 不給預設值 | 漏設就部署失敗，避免兩個 app 同時搶同一個預設 port |
| 5 | 每個 app 一個唯一 port | 目前 gentle-truths 用 9203、night-shift 用 9204，下一個用 9205 |
| 6 | Coolify 的 Domains 留空 | 填了網域，Traefik 會另外介入，並預設轉發到 port 80，造成 502 |

## 新專案檢查清單

1. 複製上面的 `compose.yaml`，改服務名稱。
2. 確認 app 監聽 `$PORT` 並綁 `0.0.0.0`。
3. Coolify 新增 Docker Compose 應用，環境變數填 `PORT=<新的唯一 port>`，Domains 留空，Deploy。
4. 在 Docker host 驗證：
   ```bash
   docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep <port>   # 應顯示 127.0.0.1:<port>->
   curl -I http://127.0.0.1:<port>                                    # 應回 200
   ```
5. 在 Cloudflare 新增 Public Hostname，指向 `http://127.0.0.1:<port>`。
6. 瀏覽器測試。

## 常見錯誤對照

| 症狀 | 原因 |
|---|---|
| `curl 127.0.0.1:PORT` 連不上 | 沒有 `ports`、`PORT` 沒設，或容器 crash（用 `docker logs` 查） |
| curl 通，但瀏覽器 502 / 1033 | Cloudflare 的 Service 填錯 port，或 `cloudflared` 沒在 host 上運行 |
| curl 回 `Connection reset` | app 只綁 `127.0.0.1`，要改綁 `0.0.0.0` |
| 部署失敗 `set PORT` | 環境變數沒設，這是預期的保護 |
| `port is already allocated` | 兩個 app 用了同一個 `PORT` |

## 前提與限制

- `cloudflared` 必須在 Docker host 本機運行，或使用 host network。若它是另一個 Docker 容器，它的 `127.0.0.1` 是它自己，連不到。這時改用 `host.docker.internal`（並加 `extra_hosts: ["host.docker.internal:host-gateway"]`），或把 `cloudflared` 與 app 放進同一個 Docker network，Service 改填 `http://<服務名>:<PORT>`。
- 此方案只適用於單一 host。要橫向擴展到多台機器時，loopback 方式就不適用。
