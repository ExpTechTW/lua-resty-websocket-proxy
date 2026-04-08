[English](#lua-resty-websocket-proxy) | [繁體中文](#lua-resty-websocket-proxy-繁體中文)

---

# lua-resty-websocket-proxy

Fork of [Kong/lua-resty-websocket-proxy](https://github.com/Kong/lua-resty-websocket-proxy) with **idle timeout** support to automatically close dead WebSocket connections.

Reverse-proxying of websocket frames with in-flight inspection/update/drop and
frame aggregation support.

Resources:

- [RFC-6455](https://datatracker.ietf.org/doc/html/rfc6455)
- [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)

# Table of Contents

- [Patch Changes](#patch-changes)
- [Synopsis](#synopsis)
- [Options](#options)
- [Limitations](#limitations)
- [License](#license)

# Patch Changes

The upstream library has a critical issue: when a WebSocket client disconnects
abnormally (network drop, browser crash, mobile network switch) **without
sending a close frame**, the proxy loop retries `recv_frame()` indefinitely on
timeout, never closing the connection. Over time this causes zombie connections
to accumulate.

### Changes from upstream

| Area | Upstream | This fork |
|------|----------|-----------|
| `_VERSION` | `0.0.1` | `0.0.1-patched` |
| Timeout behavior | Logs and continues forever | Counts consecutive timeouts; closes after threshold |
| New option: `max_idle_timeouts` | N/A | Number of consecutive timeouts before closing (default: `3`) |
| Timeout counter reset | N/A | Resets to `0` on any successful `recv_frame()` |

### How it works

```
recv_frame() timeout
       |
       v
timeout_count++
       |
       v
timeout_count >= max_idle_timeouts ?
      / \
    yes   no
     |     |
     v     v
  CLOSING  log + continue (retry recv)
  return
```

With `lua_socket_read_timeout 65s` (default in most setups) and `max_idle_timeouts = 3`,
a dead connection is closed after ~195 seconds of inactivity. Active connections
that receive any frame reset the counter to 0 and are never affected.

### Diff summary

```diff
 -- forwarder() function:
+local timeout_count = 0
 
 -- on recv_frame() timeout:
-    log(ngx.INFO, "timeout receiving frame from %s, reopening", role)
-    -- continue
+    timeout_count = timeout_count + 1
+    if max_idle_timeouts > 0 and timeout_count >= max_idle_timeouts then
+        self[self_state] = _STATES.CLOSING
+        return role, "idle timeout"
+    end
+    log(ngx.INFO, "timeout receiving frame from %s (%d/%d), reopening", ...)
+    -- continue

 -- on successful recv_frame():
+    if data then timeout_count = 0 end
```

[Back to TOC](#table-of-contents)

# Synopsis

```lua
http {
    server {
        listen 9000;

        location / {
            lua_socket_read_timeout 65s;
            lua_socket_send_timeout 65s;
            content_by_lua_block {
                local ws_proxy = require "resty.websocket.proxy"

                local proxy, err = ws_proxy.new({
                    aggregate_fragments = true,
                    max_idle_timeouts = 3,  -- close after 3 consecutive timeouts (~195s)
                    on_frame = function(proxy, role, typ, payload, last, code)
                        return payload, code
                    end
                })
                if not proxy then
                    ngx.log(ngx.ERR, "failed to create proxy: ", err)
                    return ngx.exit(444)
                end

                local ok, err = proxy:connect("ws://127.0.0.1:9001")
                if not ok then
                    ngx.log(ngx.ERR, err)
                    return ngx.exit(444)
                end

                local done, err = proxy:execute()
                if not done then
                    ngx.log(ngx.ERR, "failed proxying: ", err)
                end
            }
        }
    }
}
```

[Back to TOC](#table-of-contents)

# Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `aggregate_fragments` | boolean | `false` | Aggregate fragmented frames before invoking `on_frame` |
| `on_frame` | function | `nil` | Callback for inspecting/modifying frames |
| `recv_timeout` | number (ms) | `nil` | Override `recv_frame()` timeout per call |
| `max_idle_timeouts` | number | `3` | Consecutive timeouts before closing. Set `0` to disable (upstream behavior) |
| `client_max_frame_size` | number | `nil` | Max payload size from client |
| `client_max_fragments` | number | `nil` | Max fragment count from client |
| `upstream_max_frame_size` | number | `nil` | Max payload size from upstream |
| `upstream_max_fragments` | number | `nil` | Max fragment count from upstream |
| `debug` | boolean | `false` | Enable verbose debug logging |

[Back to TOC](#table-of-contents)

# Limitations

* Built with [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)
  which only supports `Sec-Websocket-Version: 13` (no extensions) and denotes
  its client component a
  [work-in-progress](https://github.com/openresty/lua-resty-websocket/blob/master/lib/resty/websocket/client.lua#L4-L5).

[Back to TOC](#table-of-contents)

# License

Copyright 2022 Kong Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

[Back to TOC](#table-of-contents)

---

# lua-resty-websocket-proxy (繁體中文)

基於 [Kong/lua-resty-websocket-proxy](https://github.com/Kong/lua-resty-websocket-proxy) 的分支，新增**閒置超時**功能，自動關閉死掉的 WebSocket 連線。

WebSocket 反向代理，支援即時攔截/修改/丟棄 frame 及 fragment 聚合。

參考資源：

- [RFC-6455](https://datatracker.ietf.org/doc/html/rfc6455)
- [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)

# 目錄

- [Patch 差異說明](#patch-差異說明)
- [使用範例](#使用範例)
- [選項說明](#選項說明)
- [限制](#限制)
- [授權](#授權)

# Patch 差異說明

上游原版有一個嚴重問題：當 WebSocket 客戶端異常斷線（網路中斷、瀏覽器關閉、行動網路切換）而**沒有發送 close frame** 時，proxy 的 recv loop 會在每次 timeout 後無限重試，永遠不會關閉連線。隨時間累積會產生大量殭屍連線。

### 與上游的差異

| 項目 | 上游原版 | 本分支 |
|------|----------|--------|
| `_VERSION` | `0.0.1` | `0.0.1-patched` |
| Timeout 行為 | 記錄 log 後永遠重試 | 計算連續 timeout 次數，達到門檻後關閉 |
| 新選項：`max_idle_timeouts` | 無 | 連續 timeout 幾次後關閉（預設：`3`） |
| Timeout 計數器重置 | 無 | 收到任何成功的 `recv_frame()` 後重置為 `0` |

### 運作原理

```
recv_frame() timeout
       |
       v
timeout_count++
       |
       v
timeout_count >= max_idle_timeouts ?
      / \
    是   否
     |     |
     v     v
  CLOSING  記錄 log + 繼續（重試 recv）
  return
```

以 `lua_socket_read_timeout 65s`（多數設定的預設值）搭配 `max_idle_timeouts = 3` 為例，死連線會在約 195 秒無活動後被關閉。活躍連線收到任何 frame 都會重置計數器，完全不受影響。

### Diff 摘要

```diff
 -- forwarder() 函數：
+local timeout_count = 0
 
 -- recv_frame() timeout 時：
-    log(ngx.INFO, "timeout receiving frame from %s, reopening", role)
-    -- continue
+    timeout_count = timeout_count + 1
+    if max_idle_timeouts > 0 and timeout_count >= max_idle_timeouts then
+        self[self_state] = _STATES.CLOSING
+        return role, "idle timeout"
+    end
+    log(ngx.INFO, "timeout receiving frame from %s (%d/%d), reopening", ...)
+    -- continue

 -- recv_frame() 成功時：
+    if data then timeout_count = 0 end
```

[回到目錄](#目錄)

# 使用範例

```lua
http {
    server {
        listen 9000;

        location / {
            lua_socket_read_timeout 65s;
            lua_socket_send_timeout 65s;
            content_by_lua_block {
                local ws_proxy = require "resty.websocket.proxy"

                local proxy, err = ws_proxy.new({
                    aggregate_fragments = true,
                    max_idle_timeouts = 3,  -- 連續 3 次 timeout 後關閉（約 195 秒）
                    on_frame = function(proxy, role, typ, payload, last, code)
                        return payload, code
                    end
                })
                if not proxy then
                    ngx.log(ngx.ERR, "failed to create proxy: ", err)
                    return ngx.exit(444)
                end

                local ok, err = proxy:connect("ws://127.0.0.1:9001")
                if not ok then
                    ngx.log(ngx.ERR, err)
                    return ngx.exit(444)
                end

                local done, err = proxy:execute()
                if not done then
                    ngx.log(ngx.ERR, "failed proxying: ", err)
                end
            }
        }
    }
}
```

[回到目錄](#目錄)

# 選項說明

| 選項 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `aggregate_fragments` | boolean | `false` | 在呼叫 `on_frame` 前聚合碎片化的 frame |
| `on_frame` | function | `nil` | 用於攔截/修改 frame 的回呼函數 |
| `recv_timeout` | number (ms) | `nil` | 覆寫每次 `recv_frame()` 的超時時間 |
| `max_idle_timeouts` | number | `3` | 連續超時幾次後關閉連線。設 `0` 停用（維持上游行為） |
| `client_max_frame_size` | number | `nil` | 來自客戶端的最大 payload 大小 |
| `client_max_fragments` | number | `nil` | 來自客戶端的最大 fragment 數量 |
| `upstream_max_frame_size` | number | `nil` | 來自上游的最大 payload 大小 |
| `upstream_max_fragments` | number | `nil` | 來自上游的最大 fragment 數量 |
| `debug` | boolean | `false` | 啟用詳細除錯日誌 |

[回到目錄](#目錄)

# 限制

* 基於 [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket) 建構，僅支援 `Sec-Websocket-Version: 13`（不支援擴充），其客戶端元件目前仍標記為[開發中](https://github.com/openresty/lua-resty-websocket/blob/master/lib/resty/websocket/client.lua#L4-L5)。

[回到目錄](#目錄)

# 授權

Copyright 2022 Kong Inc.

依據 Apache License, Version 2.0（「本授權」）授權；
除非遵守本授權，否則不得使用本檔案。
您可在以下位置取得本授權的副本：

   http://www.apache.org/licenses/LICENSE-2.0

除非適用法律要求或書面同意，依據本授權散布的軟體是以「現狀」為基礎散布，
不附帶任何明示或暗示的保證或條件。
請參閱本授權以瞭解特定語言規範的權限及限制。

[回到目錄](#目錄)
