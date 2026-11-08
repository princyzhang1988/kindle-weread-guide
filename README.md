# Kindle Paperwhite 1 精简改造：KOReader + 微信读书

把一台越狱 Kindle Paperwhite 1（固件 5.4.4）改造成"开机即微信读书"的专用阅读器的完整记录。

核心思路：Kindle 不是安卓系统，装不了微信读书 APK。社区方案是 **KOReader + 微信读书插件**（[finlater/weread.koplugin](https://github.com/finlater/weread.koplugin)），支持书架、进度/时长同步、划线想法，还能**免扫码用 API Key 直接登录**。

> 免责声明：仅供个人学习研究。使用微信读书第三方插件请自行评估账号风险，遵守相关用户协议。

## 设备与前提

- Kindle Paperwhite 1，固件 5.4.4
- 已越狱，已装 KUAL + MRPI + USBNetwork（参考 [KindleModding Wiki](https://kindlemodding.org/)）
- Mac（本文以 macOS 为例；Linux 用户可自行替换 networksetup 部分）

## 一、USB 网络 SSH 连接

插上数据线后 Kindle 会虚拟出一块 RNDIS 网卡。给 Mac 手动配上 USBNetwork 的标准网段：

```bash
# 找到 RNDIS/Ethernet Gadget 对应的接口名（如 en14）
networksetup -listallhardwareports

networksetup -setmanual "RNDIS/Ethernet Gadget" 192.168.15.201 255.255.255.0
```

> `192.168.15.201/244` 是 USBNetwork 的公开默认网段；如果你的 Kindle 上改过，用你自己的。

```bash
ping -c 3 192.168.15.244        # 应该通
ssh root@192.168.15.244          # 密码是 USBNetwork 默认的 toor（改过则用你的）
```

**这台老设备上 busybox 工具残缺，后面所有命令都要注意：**

| 坑 | 规避方法 |
|---|---|
| 无 `nohup` | 后台启动用子 shell 脱离：`(命令 > 日志 2>&1 &)` |
| `grep` 不支持 `-E` | 用基本正则或多个 `-e` |
| `head` 无 `-c` | 用 `dd` 或 `head -n` |
| 无 `reboot` 命令 | 用 `telinit 6` |
| eips 无抓屏功能 | 用 `cat /dev/fb0` 原始帧缓冲（见文末） |
| dropbear 不支持 SFTP | `scp` 必须加 `-O` |

## 二、安装 / 升级 KOReader

KOReader 对 PW1/PW2 都支持，直接下最新的 kindlepw2 构建（官方说明 PW1 也能用，运行时会自动识别机型）：

```bash
# 下载 https://github.com/koreader/koreader/releases 里 kindlepw2 版 zip
```

> **网络坑**：GitHub Release 大文件直连或走某些代理会 504。可靠方案是 **分块 Range 下载**：
> 每块 2MB、`curl -sL -r 起点-终点`（**必须带 -L** 跟随 302 到 release-assets CDN）、单块失败重试，最后 `cat` 合并。文件总大小可从 GitHub API 的 asset `size` 字段拿到。

上传并升级（**保留 settings/ 目录**，书架配置和书籍元数据都在里面）：

```bash
scp -O koreader-kindlepw2-vX.X.X.zip root@192.168.15.244:/mnt/us/
```

```sh
# 在 Kindle 上
kill $(pidof luajit) 2>/dev/null          # 先停掉旧版 KOReader
cd /mnt/us
mv koreader koreader.old                   # 旧版挪走，验证没问题后再删
unzip -oq koreader-kindlepw2-vX.X.X.zip    # 解压出 koreader/ 和 extensions/koreader/
cat koreader/git-rev                       # 确认版本
rm -f koreader-kindlepw2-vX.X.X.zip
```

## 三、安装微信读书插件

```bash
# 从 https://github.com/finlater/weread.koplugin/releases 下载 zip，解压上传
scp -O weread.koplugin-vX.Y.Z.zip root@192.168.15.244:/mnt/us/
```

```sh
cd /mnt/us/koreader/plugins && unzip -oq /mnt/us/weread.koplugin-vX.Y.Z.zip
```

> 插件要求 KOReader ≥ 2026.03，旧版会在「工具」菜单里找不到微信读书。

## 四、免扫码登录（API Key）

插件常规登录是微信扫码，但 **API Key 是独立的认证方式**（`Authorization: Bearer` 直连网关），可以完全跳过扫码：

1. 手机微信读书 App → **我 → 设置 → 微信读书 Skill** → 获取 API Key
2. 先停 KOReader（它的内存配置会在退出时覆盖你的改动）：

```sh
kill $(pidof luajit); sleep 5
```

3. 编辑 `/mnt/us/koreader/settings/weread.lua`，把 Key 填进 `api_key` 字段：

```lua
    ["api_key"] = "<你的微信读书API_Key>",
```

4. 重启 KOReader，**工具 → 微信读书** 即已登录，无需扫码。

验证 Key 是否有效（在任何机器上都能测）：

```bash
curl -s -X POST "https://i.weread.qq.com/api/agent/gateway" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <你的API_Key>" \
  -d '{"api_name":"/shelf/sync","skill_version":"1.0.5"}'
# 返回书架 JSON 即有效
```

> skill_version 取自插件源码 `weread/lib/protocol.lua` 里的 `WeRead.SKILL_VERSION`。

## 五、开机自启（跳过原生 Kindle 界面）

5.4.4 固件用 **upstart**（`/etc/init/*.conf`），没有可用的 `/etc/init.d`。加一个任务，等 Amazon framework 起来后自动拉起 KOReader：

```sh
mount -o remount,rw /          # 根分区默认只读
cat > /etc/init/koreader.conf << 'EOF'
# KOReader autostart
start on started framework

script
    /mnt/us/koreader/koreader.sh >> /mnt/us/ko-boot.log 2>&1
end script
EOF
mount -o remount,ro /
sync
telinit 6                       # 重启验证
```

开机约 1 分钟后 KOReader 应自动出现在屏幕上。退出 KOReader 仍可回到原生界面，下次开机又会直接进 KOReader。

## 六、封禁 OTA 更新（保命）

连着 Wi-Fi 刷微信读书，亚马逊一旦推送更新，越狱和 KOReader 全没。5.11 以下固件的封禁方法异常简单——**在用户分区根目录建一个同名文件夹**，让更新器下载不了临时文件：

```sh
mkdir -p /mnt/us/update.bin.tmp.partial
```

想恢复 OTA 时删掉这个文件夹即可（升级/降级固件前必须恢复）。

## 七、速度调优

### 1. 章节预下载（对微信读书体验提升最大）

编辑 `/mnt/us/koreader/settings/weread.lua`（同样要先停 KOReader）：

```lua
    ["cache"] = {
        ["auto_prefetch_next_chapter"] = true,   -- 默认 false
        ...
    },
```

翻到章末不再等网络。

### 2. 插件瘦身省内存（PW1 只有 256MB）

创建 `/mnt/us/koreader/settings/reader.lua`，禁用用不到的插件（保留 gestures 手势、statistics 统计、autostandby/autosuspend 待机等核心）：

```lua
-- ./settings/reader.lua
return {
    ["plugins_disabled"] = {
        ["wallabag"] = true,
        ["news_downloader"] = true,
        ["opds"] = true,
        ["calibre"] = true,
        ["terminal"] = true,
        ["texteditor"] = true,
        ["vocabbuilder"] = true,
        ["coverbrowser"] = true,
        ["kosync"] = true,
        ["exporter"] = true,
        ["japanese"] = true,
        ["hello"] = true,
        -- ...其余按需增减
    },
}
```

### 3. 清理旧版残留

升级验证通过后删掉备份目录和旧包，1.3G 用户分区能省出上百 MB。

## 八、调试技巧

**远程抓屏**（5.4.4 的 eips 没有抓屏功能，直接 dump 帧缓冲）：

```sh
cat /dev/fb0 > /tmp/fb.raw
```

```bash
scp -O root@192.168.15.244:/tmp/fb.raw .
python3 -c "
from PIL import Image
data = open('fb.raw','rb').read()
# fb 是 6 个 768x1024 的 8bpp 缓冲槽，屏幕内容在第 0 片
Image.frombytes('L',(768,1024),data[:1024*768]).save('screen.png')"
```

**SSH 后台启动 KOReader**（无 nohup）：

```sh
(/mnt/us/koreader/koreader.sh > /mnt/us/ko.log 2>&1 &)
```

**查 KOReader 运行日志**：SSH 启动时输出会重定向到 `koreader/crash.log`（不是崩溃才是 crash.log，SSH 启动方式下它就是运行日志），插件日志有 `[WeRead]` 前缀。

## 九、恢复方法

| 改动 | 恢复方式 |
|---|---|
| 开机自启 | `mount -o remount,rw / && rm /etc/init/koreader.conf && mount -o remount,ro /` |
| OTA 封禁 | 删除 `/mnt/us/update.bin.tmp.partial` |
| 插件禁用 | 删除 `settings/reader.lua` 或在 KOReader 界面里重新启用 |
| 章节预下载 | 把 `auto_prefetch_next_chapter` 改回 `false` |

## 致谢 / 参考

- [koreader/koreader](https://github.com/koreader/koreader) — 全能开源阅读系统
- [finlater/weread.koplugin](https://github.com/finlater/weread.koplugin) — 微信读书插件（另有同作者闭源版 WeReader，界面更接近官方 App）
- [KindleModding Wiki](https://kindlemodding.org/) — 越狱、KUAL、封 OTA 等权威文档
- [MobileRead 论坛](https://www.mobileread.com/forums/showthread.php?t=346037) — 越狱生态主阵地
