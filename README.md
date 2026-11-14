# astrbot_plugin_molt

让 AstrBot 成为一个**可以交出自己**的壳。

这是 [molt（蜕壳）](https://github.com/hansei996/molt) 的 AstrBot 侧插件。
它只做一件事：让外部程序能在**不杀死进程**的前提下，让机器人停止对外说话、
确认它真的安静了、再让它重新开口。

> 装它本身**不会**让机器人搬家，也不改变任何现有行为。
> 配置文件不存在时，指令层是关闭的，四个端点只报告状态。

## 它解决什么问题

要让一个机器人从 NAS 搬到另一台机器，最直接的办法是 `docker kill`。
但那把「断开一个 IM 连接」和「杀掉一个进程」绑成了同一件事 —— 代价是丢掉调度器、
定时任务、正在跑的那轮对话，恢复要重启（重拉几千条 LLM 元数据）。

这个插件提供了另一条路：**只闭嘴，不杀进程**。

## 四个端点

挂在 AstrBot 自己的 dashboard 上，走 dashboard 的 JWT 鉴权：

| 方法 | 路径 | 作用 |
|---|---|---|
| GET | `/api/plug/astrbot_plugin_molt/status` | 我是谁、连着哪些平台、WAL 有多少没合并 |
| POST | `/api/plug/astrbot_plugin_molt/drain` | 断开所有**对外**平台连接 |
| POST | `/api/plug/astrbot_plugin_molt/resume` | 把 drain 掉的重新连上（只恢复配置里 `enable=true` 的） |
| POST | `/api/plug/astrbot_plugin_molt/checkpoint` | `PRAGMA wal_checkpoint(TRUNCATE)` |

`/status` 里的 `speaking` 是唯一该被信任的结论字段 —— 别自己从 `connected` 推。
推错了会让两只机器人同时在线，而这种错误在日志里看不出来（两条都"正常运行"）。

## 聊天指令（可选，默认关闭）

在插件目录里放一个 `molt_command.json`：

```json
{
  "enabled": true,
  "shell": "nas-shell",
  "target": "win-shell",
  "data_dir": "/AstrBot/data",
  "molt_pkg": "/AstrBot/data/plugins/astrbot_plugin_molt/_vendor",
  "warden_url": "http://172.17.0.1:6186",
  "warden_token": "CHANGEME",
  "control_url": "http://127.0.0.1:6185",
  "control_config": "/AstrBot/data/cmd_config.json"
}
```

然后在 QQ 里（仅管理员）：

```
/molt go        →  搬去 <target>？120 秒内回 /molt confirm 我就动身
/molt confirm   →  好，我走了
/molt status    →  账本现在是第几代、有没有人在搬家
/molt cancel    →  取消了，我不搬
```

三道闸：**权限**（只认 `admins_id`）、**两步确认 + 120 秒时效**（提议存在内存里，
进程重启即失效）、**`enabled` 默认 false**。
漏开 = 机器人不搬（一条日志），漏关 = 它在你没盯着的时候搬了（一次事故）。

## 它依赖 molt 本体

四个端点单独装就能用；`/molt go` 那一套需要：

- NAS 侧跑着 **Warden**（账本 + 租约）
- 目标机跑着 **molt-agent**（发现新快照就落位并起 AstrBot）
- 完整流程见 [molt 仓库的 `docs/SELFUSE.md`](https://github.com/hansei996/molt/blob/main/docs/SELFUSE.md)

`_vendor/molt/` 是 molt 运行时的副本，**故意内嵌**：`data/plugins/**` 不在灵魂快照里，
所以它不会被搬来搬去；而插件需要能 import 它才能执行迁徙。

## 兼容性

- AstrBot **>= 4.24.0**，实测于 **4.29.4**
- 只使用公开接缝：`context.platform_manager`、`context.register_web_api`
- 没改 AstrBot 任何一行代码

已知边界：`drain()` 依赖 `platform_manager._inst_map` 的实例身份来判断"哪些连接是对外的"。
如果某个 AstrBot 版本改了它的内部形状，`drain()` 会退化成"关不掉"，
但**不会静默成功** —— 返回的 `ok` 会是 `false`。

## License

Apache-2.0
