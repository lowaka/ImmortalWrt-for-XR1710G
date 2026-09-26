# custom/ 定制目录说明

本仓库是 `naoki66/ImmortalWrt-for-Gemtek-XR1710G` 的 fork，所有“个性化修改”都放在这个目录，
上游（naoki66 + immortalwrt）更新时不会被覆盖。

| 文件 | 作用 |
| --- | --- |
| `config.fragment` | 插件开关（编译前追加到 `.config`，可增删要编译的插件） |
| `feeds.conf.custom` | 额外 feed（OpenClash 官方源，编译时拉取最新版） |
| `scripts/fetch-openclash-core.sh` | 编译时下载最新 mihomo 内核，打包进固件 `/etc/openclash/core/clash_meta` |
| `files/` | 固件文件覆盖层（首次开机后台地址 `192.168.5.1`、DHCP 下发 `192.168.5.100-249`；以及「状态 → 信道分析」页面修复） |

## 常用操作

- **增/减插件**：改 `custom/config.fragment`，然后手动运行 `Build XR1710G Firmware`。
- **改后台地址**：改 `custom/files/etc/uci-defaults/99-custom-network.sh`（只在首次开机或刷机后首次启动生效）。
- **同步上游**：运行 `Sync Upstream (naoki66 + ImmortalWrt)`，会自动合并并触发一次构建 + 发布 Release。

## 注意事项

- `.gitattributes` 里把 `.github/workflows/build-firmware.yml` 标记为 `merge=keep-ours`：
  同步上游时如果该文件冲突，会保留本仓库版本（因为里面含定制 hook）。上游 workflow 有大改动时，
  需要手动把新特性合并进来。
- `config.seed` 不设保护，保持跟随 naoki66 上游更新；本仓库的插件开关一律写在 `custom/config.fragment`。

## 信道分析页面修复

`files/www/luci-static/resources/view/status/channel_analysis.js` 覆盖了 luci-mod-status 安装的同名文件，
修掉 ImmortalWrt 自带「状态 → 信道分析」在这台机器上的三个问题：

| 现象 | 原因 | 修法 |
| --- | --- | --- |
| 2.4G/5G/6G 的信道柱状图全部挤在左边重叠 | 非激活标签页宽度为 0，`create_channel_graph()` 用 `offsetWidth` 算出的列宽为负 | 等面板真正有宽度后再绘制，切换标签时补绘 |
| 5GHz 只扫到低段（149/153 等 80MHz 热点不见） | 页面用 UCI 名（`radio1`）调 iwinfo，iwinfo 把它解析成该 phy 的**第一个**接口，返回的是另一个频段的结果 | 先解析真实接口名（`phy0.1-ap0`）再扫描 |
| 6GHz 一直停在 “Starting wireless scan...” | 整颗 phy 的完整扫描要 20 秒以上，超过 rpcd 的 30 秒 ubus 超时，而页面没有失败处理 | 先渲染缓存结果，再用完整扫描刷新；失败时给出提示并停止轮询 |

维护说明：

- 这是文件覆盖，不依赖 `patches/feeds/` 补丁，所以上游改动不会导致编译失败；代价是上游对同一文件的更新不会自动跟进来。
- 需要跟进上游时：`git -C feeds/luci log --oneline -- modules/luci-mod-status/htdocs/luci-static/resources/view/status/channel_analysis.js`，把上游改动手工合进本文件。
- 同样的修复已按 naoki66 的 `patches/feeds/` 约定提交 PR；若上游合并，可以直接删除这个覆盖文件。