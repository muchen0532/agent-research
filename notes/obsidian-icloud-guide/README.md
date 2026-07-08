# Obsidian + iCloud 同步 + 间隔重复复习 完整教程

用 Obsidian 记录内容（比如学习笔记），iPhone 和 Windows 电脑之间通过 iCloud 免费同步，配合 Spaced Repetition 插件实现间隔重复复习，再加一个每日提醒。全程免费，无需订阅 Obsidian Sync。

适合场景：日常积累类的记录与复习，比如学习笔记、读书摘录等，每天存一条内容，定期复习巩固；也支持多设备/多人共用同一个库。

---

## 一、iPhone 上创建 iCloud 库

1. 打开「设置」
2. 点最上面你的名字（Apple ID）
3. 点「iCloud」
4. 点「iCloud 云盘」
5. 确认最上面「iCloud 云盘」总开关是绿色（开着）
6. 往下拉，找到「Obsidian」这一行，把它的开关打开
7. 完全退出 Obsidian App（上滑关掉，不是切到后台）
8. 重新打开 Obsidian
9. 点「Create new vault」
10. 输入一个名字，比如"口语纠正"
11. 这时候应该能看到「Store in iCloud」开关，打开它
12. 点「Create」

> **常见问题**：如果第 11 步看不到「Store in iCloud」开关，大概率是第 6 步的 App 权限没生效。回到「设置 → [你的名字] → iCloud → iCloud 云盘」，检查 Obsidian 开关状态，完全退出重进 App 后再试一次。

---

## 二、Windows 电脑上安装 iCloud 并打开同一个库

1. 打开 Microsoft Store
2. 搜索 "iCloud"，安装苹果官方那个
3. 打开装好的 iCloud
4. 用手机同一个 Apple ID 登录
5. 手机上如果弹出确认提示，点确认
6. 登录后，勾选「iCloud 云盘」
7. 点「应用」，等它转圈同步完
8. 打开「文件资源管理器」
9. 左边栏找到「iCloud 云盘」，点进去
10. 找到一个叫「Obsidian」的文件夹，点进去
11. 里面能看到手机上创建的那个库

### 打开这个库

12. 打开电脑上的 Obsidian（没装先去 [obsidian.md](https://obsidian.md) 官网下载）
13. 左下角点一下头像/图标
14. 点「Manage vaults」
15. 点「Open folder as vault」
16. 找到第 11 步看到的那个库文件夹，选中，点「打开」

> 提示：iCloud 同步的是整个库文件夹，包括其中隐藏的 `.obsidian` 配置文件夹，所以电脑上装的插件、插件设置，会自动同步到 iPhone，不用在手机上重新下载插件文件。

---

## 三、安装 Spaced Repetition 插件（在电脑上装一次即可）

1. 打开 Obsidian，进入「设置 → 第三方插件」，关闭"安全模式"
2. 点击「浏览」，搜索 **Spaced Repetition**，安装并启用
3. 启用后左侧栏最下方会出现一个复习按钮，这是复习入口

### 在 iPhone 上启用同一个插件

因为插件文件已经通过 iCloud 自动同步过来，手机上只需要手动打开开关：

1. 打开手机 Obsidian
2. 设置 → 第三方插件（Community plugins）
3. 确认"安全模式"（Restricted mode）是关闭的
4. 找到列表里的 **Spaced Repetition**
5. 打开它的开关

---

## 四、怎么用

- 新建笔记，在正文里加上 `#review` 标签，这篇笔记就会自动进入复习队列
- 建议一天新建一篇笔记，标题用日期，比如 `2026-07-08`，把当天要记录的内容整段贴进去
- 点左下角的复习按钮，Obsidian 会按算法把到期的笔记推给你，看完后选择"记得/忘了"之类的反馈，它自动安排下次复习时间
