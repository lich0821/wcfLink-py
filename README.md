# wcflink

`wcflink` 是参考 [lich0821/wcfLink](https://github.com/lich0821/wcfLink) 实现的 Python 版本地运行时。

它直接对接 iLink 通道，负责扫码登录、账号持久化、长轮询收消息、本地事件存储、媒体收发，以及对外提供本地 HTTP API 和 Python SDK。

## 当前实现

- 扫码登录
- 登录状态轮询
- 已登录账号持久化
- iLink `getupdates` 长轮询
- 文本消息发送
- 图片、视频、文件、语音发送
- 图片、语音、视频、文件接收与落盘
- 本地事件存储
- `context_token` 管理
- 本地 HTTP API
- SQLite 状态存储
- Python SDK 和命令行工具

## 安装

```bash
pip install wcflink
```

运行时依赖：

- `pycryptodome`
- `qrcode[pil]`

它们会随着 `wcflink` 一起安装。

## 启动服务

最常见的用法是直接启动本地服务：

```bash
wcflink serve
```

默认监听：

```text
127.0.0.1:17890
```

默认状态目录：

```text
./data
```

也可以覆盖配置：
* 监听地址：listen-addr
* 认证密钥：api-key

注意：默认监听地址 `127.0.0.1`，如果希望监听全部使用 `0.0.0.0`，密钥全部接口都需要，添加请求头 `Authorization`，如下配置接口的值应当为 `Bearer soUY6VaaS286oiiEfr6sQJK4v3q3K020zdwC5ZIDAr8`

```bash
wcflink serve \
  --listen-addr 127.0.0.1:28080 \
  --api-key soUY6VaaS286oiiEfr6sQJK4v3q3K020zdwC5ZIDAr8 \
  --state-dir ./runtime-data \
  --upstream-base-url https://ilinkai.weixin.qq.com
```

## Python SDK

### 方式一：调用本地 HTTP API

```python
from wcflink import WcfLinkClient

client = WcfLinkClient("http://127.0.0.1:17890")

print(client.version())
print(client.list_accounts())
print(client.list_events(limit=10))
```

### 方式二：直接在进程内启动引擎

```python
from wcflink import Engine, load_config

cfg = load_config()
engine = Engine(cfg=cfg)
engine.start_background()

session = engine.start_login()
print(session.session_id)
print(session.qr_code_url)
```

## 登录示例

```python
from wcflink import WcfLinkClient

client = WcfLinkClient()

session = client.start_login()
print(session.session_id)
print(session.qr_code_url)

status = client.get_login_status(session.session_id)
print(status.status)
```

保存二维码 PNG：

```python
png = client.get_login_qr(session.session_id)
with open("qrcode.png", "wb") as f:
    f.write(png)
```

## 发送文本消息

```python
client.send_text(
    account_id="xxx@im.bot",
    to_user_id="yyy@im.wechat",
    text="hello",
)
```

注意：

- 当前发送依赖 `context_token`
- 如果 `context_token` 为空，会尝试从本地历史会话中查找
- 因此目标用户通常需要先给 bot 发过至少一条消息

## 发送媒体消息

```python
client.send_media(
    account_id="xxx@im.bot",
    to_user_id="yyy@im.wechat",
    file_path="/absolute/path/to/demo.jpg",
    media_type="image",
    text="caption",
)
```

`media_type` 当前支持：

- `image`
- `video`
- `file`
- `voice`

## HTTP API

启动服务后，可用接口与上游 Go 版保持同一思路：

- `GET /health/live`
- `GET /health/ready`
- `GET /api/version`
- `POST /api/accounts/login/start`
- `GET /api/accounts/login/status`
- `GET /api/accounts/login/qr`
- `GET /api/accounts`
- `GET /api/events`
- `GET /api/logs`
- `GET /api/settings`
- `POST /api/settings`
- `POST /api/messages/send-text`
- `POST /api/messages/send-media`

## 命令行

```bash
wcflink serve
wcflink version
wcflink accounts
wcflink events --limit 20
wcflink login start
wcflink login status login_xxx
wcflink send-text --account-id xxx@im.bot --to-user-id yyy@im.wechat --text "hello"
wcflink send-media --account-id xxx@im.bot --to-user-id yyy@im.wechat --file-path /abs/demo.jpg --type image
```

如果服务不在默认地址：

```bash
wcflink --base-url http://127.0.0.1:28080 version
```

## 构建与发布

```bash
cd wcfLink-py
python3 -m pip install --upgrade build twine
python3 -m build
python3 -m twine check dist/*
python3 -m twine upload dist/*
```

正式发布前建议确认：

- `pyproject.toml` 中的版本号已经更新
- README 在 PyPI 上能正常渲染
- 本地 `wcflink serve` 能正常启动
- 登录、轮询、发消息流程已经对真实 iLink 环境做过验证
