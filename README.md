# opencli-weixin-album

opencli plugin — 获取微信公众号合集（Album）的所有文章列表，自动下载全部文章（含图片），生成带本地路径的 Markdown 索引文件。

## 功能

- 自动获取合集全部文章链接（无需 Cookie，直接调用微信 API）
- 自动逐篇下载文章内容和图片（复用 `opencli weixin download`）
- **增量下载**：自动检测已有索引，跳过已下载的文章，只下载缺失的
- 生成 Markdown 索引文件，下载完成后自动回写本地路径
- 每次翻页/下载间隔 1-3 秒随机暂停，避免触发限流

## 前置要求

- Node.js >= 18
- [opencli](https://github.com/jackwener/opencli) >= 1.3.3
- **Chrome 浏览器** + **opencli Browser Bridge 扩展**（下载文章时需要）

```bash
npm install -g @jackwener/opencli
```

## 安装

```bash
opencli plugin install github:SlowGrowth1314/opencli-weixin-album
```

安装后插件位于 `~/.opencli/plugins/opencli-weixin-album/`。安装过程中会自动完成 npm install、依赖链接和 TypeScript 编译。

## 浏览器扩展配置（必须）

下载文章需要 opencli 通过浏览器访问微信页面，必须安装并连接 Browser Bridge 扩展：

### 1. 安装扩展

1. 打开 Chrome，访问 `chrome://extensions/`
2. 开启右上角 **Developer mode**
3. 点击 **Load unpacked**
4. 选择目录：`{node_modules}/@jackwener/opencli/extension/`
   - 全局安装路径示例：`/Users/{用户名}/.nvm/versions/node/v24.14.0/lib/node_modules/@jackwener/opencli/extension/`
5. 确认扩展显示为 **OpenCLI v1.2.6** 且已启用

### 2. 验证连接

```bash
opencli doctor
```

应该看到：

```
[OK] Daemon: running on port 19825
[OK] Extension: connected
[OK] Connectivity: connected in 0.3s
```

如果显示 `[MISSING] Extension: not connected`，请检查：
- 扩展是否已在 Chrome 中启用
- 尝试刷新扩展页面或重启 Chrome

### 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `zsh: permission denied: opencli` | `main.js` 缺少执行权限 | `chmod +x $(which opencli)` 或对 symlink 目标文件执行 `chmod +x` |
| `zsh: parse error near '&'` | URL 中的 `&` 被 shell 解析 | URL 必须用引号包裹 |
| `Package subpath './registry.js' is not defined` | import 路径带了 `.js` 后缀 | 已在新版本修复，更新插件即可 |
| `Browser Extension is not connected` | Chrome 扩展未加载或未连接 | 按上面步骤安装扩展，然后 `opencli doctor` 验证 |

## 使用方法

### 一键下载合集

在微信中打开合集页面，复制 URL，例如：

```
https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NTU3NTc5Ng==&action=getalbum&album_id=4482506796406177793&scene=21#wechat_redirect
```

执行命令（**URL 必须用引号包裹**，防止 shell 解析 `&`）：

```bash
opencli weixin download-album \
  --url "https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NTU3NTc5Ng==&action=getalbum&album_id=4482506796406177793&scene=21#wechat_redirect"
```

运行输出：

```
📦 获取合集: 4482506796406177793
📖 合集名称: 智能体设计模式
📥 4 篇 (cursor=2247484319)
✅ 共收集 4 篇文章链接
📄 已生成索引: ./weixin-albums/智能体设计模式/智能体设计模式.md

[1/4] 📥 下载: 智能体设计模式 - 第一章: 让 AI 不再「一口吃成胖子」
✅ [1/4] 下载成功，已更新本地路径: ...
[2/4] 📥 下载: 智能体设计模式-第二章: 让 AI 学会看情况办事
✅ [2/4] 下载成功，已更新本地路径: ...
...

✅ 合集下载完成: 4/4 篇
📄 索引文件: ./weixin-albums/智能体设计模式/智能体设计模式.md
```

### 指定输出目录

```bash
opencli weixin download-album \
  --url "合集URL" \
  --output ./my-articles
```

### 调整每页获取数量

```bash
opencli weixin download-album \
  --url "合集URL" \
  --batch-size 10
```

最大值为 20（微信 API 限制）。

### 增量下载（断点续传）

如果下载中断，可以直接传入已有索引文件继续下载：

```bash
opencli weixin download-album --url "./weixin-albums/智能体设计模式/智能体设计模式.md"
```

运行输出：

```
📋 增量下载模式: ./weixin-albums/智能体设计模式/智能体设计模式.md
📖 合集名称: 智能体设计模式
📊 已下载: 15 篇，待下载: 7 篇

[16/22] 📥 下载: 智能体设计模式 - 第十六章...
✅ [16/22] 下载成功
...

✅ 合集下载完成: 22/22 篇
```

**两种触发方式：**

1. **传入索引文件路径** — 直接指定 MD 文件，解析其中未下载的文章
2. **传入合集 URL** — 如果输出目录已存在索引文件，自动检测并跳过已下载的

## 输出格式

在输出目录下生成以合集名称命名的文件夹，包含索引文件和各篇文章：

```
weixin-albums/
└── 智能体设计模式/
    ├── 智能体设计模式.md                          # 索引文件
    ├── 智能体设计模式_-_第一章_让_AI_不再.../      # 第一章
    │   ├── 智能体设计模式_-_第一章_....md
    │   └── images/
    │       ├── img_001.png
    │       └── ...
    ├── 智能体设计模式-第二章_让_AI_学会.../         # 第二章
    │   ├── 智能体设计模式-第二章_....md
    │   └── images/
    └── ...
```

索引文件内容示例（本地路径列在下载完成后自动填入）：

```markdown
| # | 标题 | URL | 本地路径 | 发布时间 |
|---|------|-----|---------|---------|
| 1 | 智能体设计模式 - 第一章: 让 AI 不再「一口吃成胖子」 | https://mp.weixin.qq.com/s?... | 智能体设计模式_-_第一章.../....md | 2026-04-23 |
| 2 | 智能体设计模式-第二章: 让 AI 学会看情况办事 | https://mp.weixin.qq.com/s?... | 智能体设计模式-第二章.../....md | 2026-04-25 |
```

## 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--url` | 是 | - | 微信合集页面 URL（必须用引号包裹） |
| `--output` | 否 | `./weixin-albums` | 输出目录 |
| `--batch-size` | 否 | `20` | 每次 API 请求获取的文章数（上限 20） |

## 技术细节

- **翻页机制**：微信合集 API 使用 cursor-based 分页，通过上一页最后一条文章的 `msgid` 和 `itemidx` 作为游标请求下一页
- **文章列表无需认证**：合集文章列表为公开数据，无需 Cookie 或登录
- **文章下载需要浏览器**：通过 `opencli weixin download` 调用浏览器渲染页面获取完整内容
- **图片下载**：自动下载文章内所有图片到本地 `images/` 目录，Markdown 中图片路径替换为本地相对路径
- **限流保护**：翻页间隔 1-3 秒随机暂停

## License

MIT


# Windows 下下载微信公众号合集文章操作指南

## 目标

下载微信公众号合集文章，并将每篇文章保存为 Markdown 文件。

适用场景：

* 微信公众号合集链接形如：

```text
https://mp.weixin.qq.com/mp/appmsgalbum?__biz=...&album_id=...
```

* 使用 `opencli-weixin-album` 可以成功获取合集文章列表；
* 但直接批量下载时可能在 Windows 上报错，或误判所有文章已经下载完成。

---

## 一、准备环境

### 1. 安装 OpenCLI 和插件

```bat
npm install -g @jackwener/opencli
opencli plugin install github:SlowGrowth1314/opencli-weixin-album
```

### 2. 检查 Chrome Browser Bridge 是否正常

运行：

```bat
opencli doctor
```

正常结果应看到类似：

```text
[OK] Daemon: running
[OK] Extension: connected
[OK] Connectivity: connected
Everything looks good!
```

只要 `Extension: connected`，说明 Chrome 扩展连接正常。

---

## 二、确认单篇文章可以下载

先找一篇合集里的文章链接，测试单篇下载：

```bat
opencli weixin download --window foreground --url "单篇文章URL" --output ".\weixin-test" -v
```

如果成功，会看到类似：

```text
Status: success
Saved: weixin-test\...\文章标题.md
```

这一步很关键。

如果单篇成功，说明：

* Chrome 扩展正常；
* 微信文章能访问；
* 登录态/访问环境没问题；
* 后续失败主要是合集插件批量调用的问题。

---

## 三、用合集插件生成索引文件

运行合集下载命令：

```bat
opencli weixin download-album --url "https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzYzMTg3OTAyMQ==&action=getalbum&album_id=4514218896866000901&scene=126&sessionid=1782734528982#wechat_redirect"
```

如果正常，会看到类似：

```text
📦 获取合集: 4514218896866000901
📖 合集名称: 医生手里的数据如何变成论文
📥 20 篇
📥 40 篇
📥 41 篇
✅ 共收集 41 篇文章链接
📄 索引文件: C:\Users\a2785\weixin-albums\医生手里的数据如何变成论文\医生手里的数据如何变成论文.md
```

此时重点是生成了索引文件：

```text
C:\Users\a2785\weixin-albums\医生手里的数据如何变成论文\医生手里的数据如何变成论文.md
```

如果插件后面批量下载失败，可以停止，不影响后续操作。

---

## 四、不要直接用索引文件续跑插件

不要运行：

```bat
opencli weixin download-album --url "C:\Users\a2785\weixin-albums\医生手里的数据如何变成论文\医生手里的数据如何变成论文.md"
```

原因：

文章标题里可能含有 `|`，而插件在增量模式解析 Markdown 表格时可能会错位，把文章误判成已经下载完成。

典型表现是：

```text
已下载: 41 篇，待下载: 0 篇，共 41 篇
全部文章已下载完成，无需继续
```

但实际上本地并没有 41 篇正文文件。

---

## 五、推荐做法：用 PowerShell 一行命令批量下载正文

在 CMD 里直接运行下面这一整行，不需要手动换行：

```bat
powershell -NoProfile -ExecutionPolicy Bypass -Command "$Index=$env:USERPROFILE+'\weixin-albums\医生手里的数据如何变成论文\医生手里的数据如何变成论文.md'; $Out=$env:USERPROFILE+'\weixin-redownload'; $env:OPENCLI_WINDOW='foreground'; $env:OPENCLI_BROWSER_COMMAND_TIMEOUT='120'; New-Item -ItemType Directory -Force -Path $Out | Out-Null; $text=Get-Content -LiteralPath $Index -Raw -Encoding UTF8; $urls=[regex]::Matches($text,'https?://mp\.weixin\.qq\.com/s\?[^ \t\r\n|)]+') | ForEach-Object { $_.Value -replace '&amp;','&' } | Select-Object -Unique; Write-Host ('共找到 '+$urls.Count+' 个文章链接'); $i=0; foreach($url in $urls){ $i++; Write-Host ('['+$i+'/'+$urls.Count+'] 下载：'+$url); & opencli weixin download --window foreground --url $url --output $Out -v; if($LASTEXITCODE -ne 0){ Add-Content -LiteralPath (Join-Path $Out 'failed.txt') -Value $url; Write-Host '失败，已记录到 failed.txt' }; Start-Sleep -Seconds 3 }; Get-ChildItem $Out -Recurse -Filter *.md | Measure-Object"
```

这条命令会：

1. 读取合集索引文件；
2. 从索引里提取所有 `mp.weixin.qq.com/s?...` 文章链接；
3. 逐篇调用：

```bat
opencli weixin download --window foreground --url "文章URL" --output "输出目录" -v
```

4. 每篇之间暂停 3 秒；
5. 失败的文章会记录到：

```text
C:\Users\a2785\weixin-redownload\failed.txt
```

6. 最后统计已下载的 Markdown 文件数量。

---

## 六、打开下载目录

下载完成后运行：

```bat
explorer "%USERPROFILE%\weixin-redownload"
```

正常情况下，每篇文章会保存成一个独立文件夹，里面包含：

```text
文章标题.md
images\
```

如果合集有 41 篇文章，最终应接近 41 个正文 Markdown 文件。

---

## 七、检查下载数量

运行：

```bat
powershell -NoProfile -Command "Get-ChildItem \"$env:USERPROFILE\weixin-redownload\" -Recurse -Filter *.md | Measure-Object"
```

如果输出数量接近合集文章数，说明下载完成。

例如：

```text
Count    : 41
```

---

## 八、常见问题

### 1. 报错：`invalid album URL or index path`

原因是把单篇文章链接传给了 `download-album`。

错误示例：

```bat
opencli weixin download-album --url "https://mp.weixin.qq.com/s?..."
```

正确做法：

* 合集下载用：

```bat
opencli weixin download-album --url "https://mp.weixin.qq.com/mp/appmsgalbum?...album_id=..."
```

* 单篇文章下载用：

```bat
opencli weixin download --url "https://mp.weixin.qq.com/s?..."
```

---

### 2. 报错：`spawn EINVAL`

这是 Windows 下插件内部调用子进程的问题。

解决办法：

不要依赖插件批量下载正文，改用上面的 PowerShell 一行命令，逐篇调用已经验证成功的单篇下载命令。

---

### 3. 报错：`'mid' 不是内部或外部命令`

这是因为微信文章 URL 里有很多 `&mid=...&idx=...&sn=...` 参数。

在 CMD 的 shell 环境里，`&` 会被当成命令分隔符。

解决办法：

不要手动拼接不带引号的 URL；用 PowerShell 脚本里的：

```powershell
& opencli weixin download --window foreground --url $url --output $Out -v
```

这样 PowerShell 会把 `$url` 当成一个完整参数传给 OpenCLI。

---

### 4. 显示 downloaded，但本地没文件

这是插件增量模式解析索引时可能误判。

原因通常是文章标题里含有 `|`，导致 Markdown 表格列错位。

解决办法：

不要用索引文件继续跑 `download-album`，而是用 PowerShell 从索引里提取真实 URL，再逐篇下载。

---

## 九、最终推荐流程

完整流程如下：

```bat
opencli doctor
```

确认正常后：

```bat
opencli weixin download-album --url "合集URL"
```

生成索引文件后，不管插件后续批量下载是否失败，直接用 PowerShell 一行命令批量下载正文。

最后检查：

```bat
powershell -NoProfile -Command "Get-ChildItem \"$env:USERPROFILE\weixin-redownload\" -Recurse -Filter *.md | Measure-Object"
```

并打开目录：

```bat
explorer "%USERPROFILE%\weixin-redownload"
```

---

## 十、本次成功路径总结

本次最终跑通的关键点是：

1. `opencli doctor` 显示 daemon、Chrome extension、connectivity 全部正常；
2. 单篇文章 `opencli weixin download` 可以成功；
3. `opencli-weixin-album` 能抓取合集 41 篇文章链接；
4. 插件在 Windows 下批量下载阶段不稳定，因此绕开它；
5. 使用 PowerShell 从合集索引中提取文章 URL；
6. 逐篇调用 `opencli weixin download`，最终成功下载正文。
