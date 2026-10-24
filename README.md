# 小米电视去广告 + 换开源桌面（FLauncher）完全攻略

> 适用于国内版 MIUI TV（实测机型 MiTV-ASTP0 / Android 9，2026-10 成功）
> 免 root、全程可逆（所有操作都有对应还原命令）。
> 原理：ADB 调试 + `pm uninstall --user 0`（只对当前用户卸载，系统分区不动，
> `cmd package install-existing <包名>` 一条命令可还原）。

## 成果

| 项 | 结果 |
|---|---|
| 广告/推送/统计组件 | 全部移除，重启存活 |
| 开机桌面 | PatchWall（瀑布流+广告）→ **FLauncher**（开源极简） |
| HOME 键 | 始终回到 FLauncher |
| 系统设置 | 不受影响（`com.xiaomi.mitv.settings`） |

---

# Part 1 · 去广告（基础篇）

## 一、解锁开发者模式并开启 ADB

1. **设置 → 关于 → 「产品型号」**（或「版本号」）**连按遥控器"确定"5-7 次** → 提示"已开启开发者模式"
2. **设置 → 账号与安全 → ADB 调试 → 允许**（部分固件在 设置 → 通用设置 → 开发者选项）
3. 首次连接时电视会弹 RSA 授权框，遥控器选"允许"（勾选"一律允许"）

> 注意：部分固件（如 MiTV-ASTP0）**每次重启 ADB 开关会被复位**，需重新开启。
> 没有桌面时可用语音键说"打开设置"进入。

## 二、电脑端连接

```bash
adb connect <电视IP>:5555
adb devices   # 确认 device 状态（unauthorized=电视上有弹窗没点）
```

电视与电脑同一路由器即可（2.4G/5G 任意，同网段互通；不要用访客网络）。

## 三、去广告组件清单（按固件实测调整）

社区通用清单（部分固件只有其中几项）：

| 包名 | 作用 |
|---|---|
| `com.miui.systemAdSolution` | 系统广告引擎（横幅/开屏，核心） |
| `com.duokan.videodaily` | 今日头屏 |
| `com.xiaomi.mitv.tvpush`(.tvpushservice) | 消息推送弹窗 |
| `com.xiaomi.mitv.advertise` | 广告服务 |
| `com.mitv.tvad` | 电视广告 |
| `com.mitv.analysis` / `com.miui.analytics` / `com.miui.tv.analytics` | 行为统计 |
| `com.xiaomi.mistat.analyticstracker` | 埋点上报 |
| `com.xiaomi.tv.musicadapter` | 音乐卡片 |
| `com.xiaomi.mitv.tvmanager` | 电视管家 |
| `com.mitv.shoplugin` | 商城插件 |

```bash
adb shell pm uninstall --user 0 com.miui.analytics
# ...逐个执行；先 pm list packages 确认存在再删
```

**不要碰**：`com.android.packageinstaller`、`com.mitv.tvplayer` 等核心组件。
**OTA 升级可能复活组件**：重跑一遍即可。

## 四、路由器层辅助（可选）

屏蔽 `ad.xiaomi.com`、`ad1.xiaomi.com`、`ad.mi.com`。

---

# Part 2 · 换桌面（进阶篇 · 逆向实录）

## 目标

开机不进 PatchWall，直接进 [FLauncher](https://gitlab.com/flauncher/flauncher)（开源、零广告、零推荐）。

## 踩坑记录（排除法）

| 尝试 | 结果 |
|---|---|
| `cmd package set-home-activity` | 返回 Success 但**无效**（首选记录写入了，解析不认） |
| `pm uninstall --user 0 com.mitv.tvhome` | 黑屏 → 落在系统兜底页 FallbackHome |
| `settings put system tvservice_forground_app` | 该键与桌面解析无关 |
| 重打包 FLauncher 加 `android:priority="1000"` | PMS 解析后 priority 仍为 **0**（被钳制） |
| `setprop sys.fallbackhome_exit` | 无 root 不可写 |

## 逆向过程

工具链（全部本地可复现）：

1. **`vdexExtractor`**（源码编译）——这固件 `framework.jar`/`services.jar` 是 183 字节壳，真代码在：
   - `/system/framework/boot-framework.vdex`（21MB，framework）
   - `/system/framework/oat/arm/services.vdex`（10MB，system_server）
2. 提取得到 cdex（compact dex）→ baksmali 2.5.2 **不支持 cdex001** → 改用 `vdexExtractor --dis` 内置反汇编（输出 386MB 日志）
3. `jadx` 反编译普通 APK（TvSettings.apk 里的 FallbackHome 是标准 dex）

## 根因（三重夹击）

1. **AOSP 规则**：`set-home-activity` 写入的首选记录 `mAlways=false`；当其它候选的查询 match（0x108000）高于首选记录的 mMatch（0x100000）时，**首选被跳过**
2. **MIUI PMS 补丁**：第三方桌面 HOME intent-filter 的 priority 被钳制为 0（manifest 写 1000 也无效）
3. **兜底页**：`com.android.tv.settings/.system.FallbackHome`（系统特权应用）在 PatchWall 缺席时永远赢得解析；它自己的逻辑是"解析结果不是我就每 20 秒重试"——死循环黑屏

关键证据：

```
$ adb shell cmd package resolve-activity -a android.intent.action.MAIN -c android.intent.category.HOME
priority=-1000 match=0x108000 isDefault=true
com.android.tv.settings.system.FallbackHome      # PatchWall 被禁时

$ adb shell dumpsys package | grep -A6 'Preferred Activities'
  me.efesser.flauncher/.MainActivity
   mMatch=0x100000 mAlways=false                  # 首选在但被规则跳过
```

## 最终解法 ✅

```bash
adb connect <TV_IP>:5555
# 1. 装 FLauncher（原版即可，priority 补丁非必需）
adb install flauncher-armeabi-v7a-release.apk
# 2. 注册首选桌面
adb shell cmd package set-home-activity me.efesser.flauncher/me.efesser.flauncher.MainActivity
# 3. 禁用 PatchWall（保留在系统分区，可随时还原）
adb shell pm uninstall --user 0 com.mitv.tvhome
# 4. 关键一步：禁用兜底页宿主（AOSP 设置包）
adb shell pm disable-user --user 0 com.android.tv.settings
# 5. 验证：HOME 键应落在 FLauncher
adb shell input keyevent 3 && adb shell dumpsys window windows | grep mCurrentFocus
```

> 第 4 步安全性说明：国内 MIUI TV 用户看到的中文「设置」是 `com.xiaomi.mitv.settings`，
> AOSP 的 `com.android.tv.settings` 只承载兜底页（和英文设置入口），禁用不影响日常使用。

## 一键回滚

```bash
adb shell cmd package install-existing com.mitv.tvhome   # 恢复 PatchWall
adb shell pm enable com.android.tv.settings               # 恢复兜底页
adb shell pm uninstall me.efesser.flauncher               # （可选）卸载 FLauncher
```

## 其它注意事项

- **ADB 每次重启被复位**：语音"打开设置" → 账号与安全 → ADB 调试
- FLauncher 换卡顿可换当贝桌面/Projectivy，但**当贝桌面闭源带推荐**，介意请用 FLauncher
- 开机广告由固件启动流程加载，删 `com.mitv.tvad`（在装的前提下）多数固件可跳过

## 参考项目

- [MiharaKinu/MiTVNoAd](https://github.com/MiharaKinu/MiTVNoAd)（国内版专用）
- [AlPeFe/xiaomi-tv-debloat](https://github.com/AlPeFe/xiaomi-tv-debloat)（国际版，+602MB 内存实测）
- [HridayAgg/mitv-debloat](https://github.com/HridayAgg/mitv-debloat)（只卸不删原则）
- [cleanyourtv.site 小米机型清单](https://cleanyourtv.site/brand/xiaomi/)
- [博客园 ADB 实战](https://www.cnblogs.com/lamtisa/p/13624544.html) / [CSDN 开机广告清除](https://blog.csdn.net/qq_35989688/article/details/126525833)
