# weread.koplugin 离线时长补报补丁

基于 [finlater/weread.koplugin](https://github.com/finlater/weread.koplugin) **v1.7.0** 的增量补丁，为「阅读时长上报」增加**离线记账 + 联网自动补报**能力（原版离线时的阅读时长会被直接丢弃）。

## 解决什么问题

原版插件的时长上报（每 30/60 秒一条）依赖两个条件：**有网络** + **有有效的 web 登录会话（cookie）**。任一不满足，当次时长直接丢弃。这个补丁把它变成：

| 场景 | 原版 | 补丁后 |
|---|---|---|
| 在线 + cookie 有效 | 正常上报 | 正常上报（不变） |
| 离线 | **时长丢弃** | 记入本地持久化队列 |
| 从未登录 / cookie 过期 | **时长丢弃** | 同样记入队列，不丢 |
| 重新联网 / 扫码登录后 | — | 积压时长自动分块补报 |

## 工作原理

- **记账**：上报 tick 遇到"离线 / 未登录 / cookie 过期"时，把本次 interval 的时长和当前书、章节、位置存入 `weread.lua` 设置文件的 `offline_read_sessions` 队列。同书同章的连续离线 tick 会合并成一条，队列按 `offline_queue_max_hours`（默认 168 小时阅读量）裁剪最旧的。
- **补报**：收到 KOReader 的 `NetworkConnected` 事件、插件启动 20 秒后、设备从休眠唤醒、或在菜单里手动触发时，按 **FIFO、每块 ≤5 分钟、块间隔 2 秒、单次会话总量 ≤6 小时** 的节奏回放队列。只有服务端确认接受的块才会从队列扣除；认证失败会把队列原样保留并在状态页提示重新扫码。
- **复用原管道**：回放完全走插件现成的 `_run_pipeline`（含上下文刷新、cookie 续期、子进程隔离），不改任何协议/签名逻辑。

## 安装

把本目录下的文件按相同相对路径覆盖到 KOReader 的插件目录：

```
koreader/plugins/weread.koplugin/
├── main.lua                      ← 覆盖
└── weread/
    ├── lib/
    │   ├── read_report.lua       ← 覆盖
    │   ├── settings.lua          ← 覆盖
    │   └── reader_lifecycle.lua  ← 覆盖
    └── ui/
        └── read_report.lua       ← 覆盖
```

覆盖前建议备份原文件；覆盖后重启 KOReader。可先用设备自带的 luajit 做语法校验：

```sh
/mnt/us/koreader/luajit -e "assert(loadfile('/mnt/us/koreader/plugins/weread.koplugin/weread/lib/read_report.lua'))" && echo OK
```

## 使用

1. 完成一次微信扫码登录（上报走 web 会话认证，这是硬前提；之后 cookie 自动续期）
2. 工具 → 微信读书 → 设置 → **Enable reading time report** 开启，模式选 Auto-associate（或手动指定书）
3. 正常阅读即可；离线读的时长会自动积压，联网后自动补报
4. 菜单里新增了 **Flush offline reading time**（立即补报）入口；Report status 状态页会显示「Offline pending: X min」待补分钟数，cookie 失效时会提示重新扫码

## 注意

- ⚠️ 插件自带的「在线更新」会**覆盖本补丁**——更新插件后需要重新覆盖一次补丁文件（或先不在线更新，等新版本出来对照重打）
- 补报请求的时间戳只能是发送时刻（协议限制），无法还原"当时在读"的真实时间点；补丁用分块 + 限速来降低风控风险，但请自行评估使用第三方上报的风险
- 本补丁只做**时长**补报；阅读进度的离线补传不在范围内
- 修改的文件基于 v1.7.0；其他版本需对照重打
