# 影视仓 / OK影视(FongMi) / TVBox：配置与坑

三个都是 TVBox 配置方言家族：同一个配置 URL 通吃。

| App | 形态 | 获取 |
|---|---|---|
| TVBox OSC | 瀑布流（老牌，2024 停更边缘） | 各聚合仓库 |
| [FongMi(OK影视)](https://github.com/FongMi/Release) | 搜索型，维护最勤 | `leanback-armeabi_v7a.apk` |
| 影视仓 | 瀑布流 + 站点墙 | 社区聚合仓库搜 "影视仓 TV 32位" |

## 推荐配置源（免电脑免代理，每日自动更新）

```
https://fastly.jsdelivr.net/gh/hebijunge/tvbox-config@main/tvbox_recommended.json
```

（GitHub Actions 每日聚合 1630 站 + 571 直播 + 健康检查；jsdelivr 国内直连）
验证配置有效性：JSON 含 `sites[]` 且数量 > 100。

## 配置注入流程（以影视仓为例，全家族通用）

1. 装好 App → 首启过免责声明（**组合键：`下 → 右 → 确定`**，纯点击无效）
2. 进「配置」页 → 焦点放名字框
3. 按 [remote-toolbox](remote-toolbox.md) 注入：
   - 名字框：`CDN聚合`；链接框：上面的 URL（**TAB 键在字段间移动，别信坐标**）
4. `TAB TAB → 左移 → 确定键` 保存（D-pad 激活，见 toolbox）
5. 重启 App 验证站点加载

## 必知坑

| 坑 | 解法 |
|---|---|
| **聚合配置第一个站可能是成人站** | 装好后立刻按**遥控器菜单键**(keyevent 82)呼出"选择首页数据源"，切常规站（如"无广┃[采集]"） |
| `pm clear` 后"确定"按不动 | 存储权限丢了：`pm grant <pkg> android.permission.READ_EXTERNAL_STORAGE` + WRITE |
| 饭太硬等接口域名失效 | 先诊断：`getent hosts www.饭太硬.com` 若解析到 157.240.x.x（Facebook 段）= DNS 污染，换源别硬等 |
| 想给 mytv 等直播 App 喂源 | 配置里的 `lives` 段可导出为 txt/m3u 单独用 |

## 存储权限一键补

`scripts/mitv-debloat.sh perm <TV_IP>`（覆盖 TVBox/FongMi/影视仓）
