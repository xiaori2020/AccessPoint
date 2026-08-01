#AccessPoint Go 插件开发指南

AccessPoint 使用独立进程插件。每个插件是一个单独的可执行程序，通过标准输入、标准输出上的 NDJSON 协议与主程序通信。插件崩溃不会直接使 AccessPoint 进程崩溃。

## 1. 插件目录

AccessPoint 在当前工作目录的 `plugins/` 中查找插件：

```text
plugins/
└── hello/
    ├── plugin.json
    ├── hello-plugin
    └── data/
```

Windows 下入口可以写成 `hello-plugin.exe`。

每个一级子目录代表一个插件，`plugin.json` 是插件清单。`data/` 由加载器创建，权限在支持的系统上设置为 `0700`。

## 2. 创建插件

建立 Go 程序：

```go
package main

import (
    "fmt"
    "os"

    "github.com/ATStore/AccessPoint/plugin/sdk"
)

type Plugin struct {
    sdk.BasePlugin
}

func (Plugin) Metadata() sdk.Metadata {
    return sdk.Metadata{
        Name:       "hello",
        Author:     "your-name",
        Version:    "0.1.0",
        APIVersion: sdk.APIVersion,
        Permissions: []string{
            sdk.ActionLog,
            sdk.ActionSay,
        },
    }
}

func (Plugin) OnActive(_ sdk.Context, event sdk.ActiveEvent) ([]sdk.Action, error) {
    return []sdk.Action{
        sdk.MustAction(sdk.ActionLog, sdk.LogAction{
            Level: "success",
            Message: "已连接服务器 " + event.ServerCode,
        }),
    }, nil
}

func main() {
    if err := sdk.Run(Plugin{}); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

`sdk.BasePlugin` 为所有事件提供空实现，只需覆盖实际使用的回调。

## 3. 插件生命周期

插件接口：

```go
type Plugin interface {
    Metadata() Metadata
    OnPreload(Context, PreloadEvent) ([]Action, error)
    OnActive(Context, ActiveEvent) ([]Action, error)
    OnChat(Context, ChatEvent) ([]Action, error)
    OnPlayerJoin(Context, PlayerEvent) ([]Action, error)
    OnPlayerLeave(Context, PlayerEvent) ([]Action, error)
    OnStop(Context, StopEvent) ([]Action, error)
}
```

事件顺序：

1. `OnPreload`：插件进程启动、数据目录创建后触发；此时还没有游戏连接。
2. `OnActive`：AccessPoint 完成游戏接入后触发。
3. `OnChat`：收到玩家聊天时触发，必须在清单的 `events` 中声明 `chat`。
4. `OnPlayerJoin` / `OnPlayerLeave`：玩家变化时触发，分别声明 `player_join`、`player_leave`。
5. `OnStop`：AccessPoint 退出、连接关闭或加载失败清理时触发。

原始数据包事件默认不开放。

## 4. plugin.json

```json
{
  "name": "hello",
  "author": "your-name",
  "version": "0.1.0",
  "api_version": 1,
  "description": "示例插件",
  "entry": "hello-plugin",
  "enabled": true,
  "events": ["chat", "player_join", "player_leave"],
  "permissions": ["plugin.log", "game.say"],
  "timeout_seconds": 5
}
```

字段：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `name` | 是 | 插件名 |
| `author` | 否 | 作者 |
| `version` | 是 | 插件版本 |
| `api_version` | 是 | 当前必须为 `1` |
| `entry` | 是 | 插件目录内的可执行文件，相对路径 |
| `enabled` | 是 | 是否加载 |
| `events` | 否 | 订阅事件 |
| `permissions` | 否 | 允许返回的动作 |
| `timeout_seconds` | 否 | 单事件超时，默认 5 秒，最大 30 秒 |

`entry` 不允许绝对路径或 `..` 路径逃逸。

## 5. 权限与动作

### plugin.log

```go
sdk.MustAction(sdk.ActionLog, sdk.LogAction{
    Level: "info",
    Message: "插件日志",
})
```

支持等级：`info`、`success`、`warning`、`error`。

### game.say

```go
sdk.MustAction(sdk.ActionSay, sdk.SayAction{
    Target: "玩家名",
    Message: "你好",
})
```

`Target` 留空时使用机器人发言；不为空时向指定目标发送消息。

### game.command

```go
sdk.MustAction(sdk.ActionCommand, sdk.CommandAction{
    Command: "say hello",
})
```

必须在清单中声明 `game.command`。命令前导 `/` 可有可无。

AccessPoint 不向插件提供验证 Token、Cookie、密码、`sessionid` 或 `chainInfo`。

## 6. 事件示例

聊天事件：

```go
func (Plugin) OnChat(_ sdk.Context, event sdk.ChatEvent) ([]sdk.Action, error) {
    if event.Message != "hello" {
        return nil, nil
    }
    return []sdk.Action{
        sdk.MustAction(sdk.ActionSay, sdk.SayAction{
            Target: event.Player,
            Message: "你好，" + event.Player,
        }),
    }, nil
}
```

玩家事件：

```go
func (Plugin) OnPlayerJoin(_ sdk.Context, event sdk.PlayerEvent) ([]sdk.Action, error) {
    return []sdk.Action{
        sdk.MustAction(sdk.ActionLog, sdk.LogAction{
            Message: event.Name + " 加入了游戏",
        }),
    }, nil
}
```

## 7. 构建和安装

以示例插件为例：

```bash
cd /www/AccessPoint
mkdir -p plugins/hello
go build -o plugins/hello/hello-plugin ./examples/plugins/hello
cp examples/plugins/hello/plugin.json plugins/hello/plugin.json
chmod 755 plugins/hello/hello-plugin
```

然后正常启动 AccessPoint。启动日志会显示：

```text
插件已加载：hello v0.1.0 · xiaori
```

跨平台构建：

```bash
GOOS=linux GOARCH=amd64 go build -o hello-plugin ./examples/plugins/hello
GOOS=windows GOARCH=amd64 go build -o hello-plugin.exe ./examples/plugins/hello
GOOS=darwin GOARCH=arm64 go build -o hello-plugin ./examples/plugins/hello
```

## 8. NDJSON 协议

SDK 已封装协议，普通插件不需要手写。每行是一条 JSON 消息。

主程序发送：

```json
{"id":"1","type":"active","payload":{"server_code":"44987498"}}
```

插件响应：

```json
{"id":"1","ok":true,"actions":[{"type":"plugin.log","payload":{"level":"success","message":"ready"}}]}
```

标准输出只能写协议 JSON。普通调试文本必须写到标准错误：

```go
fmt.Fprintln(os.Stderr, "debug message")
```

标准错误会以 `[插件:插件名]` 的形式进入 AccessPoint 日志。

## 9. 安全说明

独立进程可以隔离崩溃，但不是完整沙箱：

- 插件仍拥有 AccessPoint 所在系统用户的文件和网络权限。
- 只安装可信插件。
- 不要让 AccessPoint 以 root 运行第三方插件。
- 生产环境建议使用独立低权限用户、容器、systemd 沙箱或其他操作系统隔离。
- 清单权限只限制 AccessPoint API 动作，无法阻止插件直接访问操作系统资源。
- 插件响应上限为 1 MiB；单事件超时后该事件失败。
- 原始 Minecraft 数据包监听默认关闭。

## 10. 从零创建和验证插件

以 `hello` 为例，在项目根目录执行：

```bash
mkdir -p examples/plugins/hello
```

至少创建：

```text
examples/plugins/hello/
├── main.go
└── plugin.json
```

开发期间依次执行：

```bash
gofmt -w examples/plugins/hello/main.go
go test ./examples/plugins/hello
go vet ./examples/plugins/hello
```

安装到 Linux 客户端：

```bash
mkdir -p plugins/hello
go build -trimpath -o plugins/hello/hello-plugin ./examples/plugins/hello
cp examples/plugins/hello/plugin.json plugins/hello/plugin.json
chmod 755 plugins/hello/hello-plugin
```

无需连接游戏服务器即可模拟 NDJSON 事件：

```bash
printf '%s\n' \
  '{"id":"1","type":"preload","payload":{"plugin_dir":"./plugins/hello","data_dir":"./plugins/hello/data"}}' \
  '{"id":"2","type":"chat","payload":{"player":"Steve","message":"hello"}}' \
  | ./plugins/hello/hello-plugin
```

每个输入事件必须得到一行 JSON 响应，响应 `id` 必须和事件一致。若标准输出混入调试文本，加载器会把插件判定为协议错误。

## 11. 玩家封禁插件

项目包含可运行示例：

```text
examples/plugins/player-ban/
plugins/player-ban/
```

功能命令：

```text
!ban <玩家名> [原因]
!unban <玩家名>
!banlist
!banhelp
```

首次模拟或实际加载后会生成：

```text
plugins/player-ban/data/config.json
plugins/player-ban/data/bans.json
```

默认管理员列表为空，所有封禁管理命令都会拒绝执行。停止 AccessPoint 后编辑 `data/config.json`：

```json
{
  "_version": 1,
  "admins": ["你的游戏名"],
  "command_prefix": "!",
  "default_reason": "违反服务器规则",
  "kick_message": "你已被服务器封禁：{reason}"
}
```

重启 AccessPoint 使配置生效。管理员名称按游戏内实际名称填写；不要把 `data/config.json` 和 `data/bans.json` 提交到公共仓库。

重新构建和安装：

```bash
go test -race ./examples/plugins/player-ban
go vet ./examples/plugins/player-ban
go build -trimpath -o plugins/player-ban/player-ban-plugin ./examples/plugins/player-ban
cp examples/plugins/player-ban/plugin.json \
   examples/plugins/player-ban/config.default.json \
   examples/plugins/player-ban/config.schema.json \
   plugins/player-ban/
chmod 755 plugins/player-ban/player-ban-plugin
```

## 12. 跨平台插件

插件是独立可执行程序，必须与客户端目标平台和架构匹配：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -o hello-plugin ./examples/plugins/hello

CGO_ENABLED=0 GOOS=windows GOARCH=amd64 \
  go build -trimpath -o hello-plugin.exe ./examples/plugins/hello

CGO_ENABLED=0 GOOS=android GOARCH=arm64 \
  go build -trimpath -o hello-plugin ./examples/plugins/hello
```

Windows 清单的 `entry` 要指向 `.exe` 文件。Android 产物面向 Termux 命令行环境，不是 APK。

## 13. 示例代码

完整示例位于：

```text
examples/plugins/hello/main.go
examples/plugins/hello/plugin.json
examples/plugins/player-ban/main.go
examples/plugins/player-ban/main_test.go
examples/plugins/player-ban/plugin.json
```
