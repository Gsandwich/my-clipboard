<div align="center">
  <img src="src-tauri/icons/128x128.png" width="96" height="96" alt="剪贴板图标" />
  <h1>Clipboard Manager</h1>
  <p><strong>复制过的东西，回头还找得到。</strong></p>
  <p>一个本地优先的 Windows 剪贴板管理器：保存文字与图片历史，分类、搜索、一键复制，附带一个本地密码本。</p>

  <p>
    <img alt="Windows 10 和 11" src="https://img.shields.io/badge/Windows-10%20%7C%2011-2d6f8e?logo=windows11&logoColor=white" />
    <img alt="Tauri 2" src="https://img.shields.io/badge/Tauri-2-24C8DB?logo=tauri&logoColor=white" />
    <img alt="Svelte 5" src="https://img.shields.io/badge/Svelte-5-FF3E00?logo=svelte&logoColor=white" />
    <img alt="License MIT" src="https://img.shields.io/badge/license-MIT-5c554d" />
  </p>
</div>

## 它解决什么问题

Windows 自带的 `Win + V` 只保留最近几十条，记录一多就没有办法了：找不到、不能分类、图片管理不了、也没法把一段富文本转成纯文本再复制。

这个工具把「找回来、整理好、再复制」这三件事放进一个常驻托盘的小窗口里：

- 按 **Alt + C** 随时唤出，用完点 × 收进托盘（不是退出）
- 文字和图片分开放，各自有历史列表
- 自定义分类，搜索框即时过滤
- 想真退出：右下角托盘图标右键 → Quit

## 功能

**剪贴板历史**

- 自动记录复制的文字和图片，图片保存为缩略图 + 原图
- 分类标签（默认「全部」，可自建、重命名、删除）
- 搜索框按内容过滤，中英文都支持
- 点一下即复制回剪贴板，右键菜单可预览、删除
- 文字条目 hover 才显示操作按钮，减少视觉噪声

**密码本**

- 按用途记录账号密码，支持显示/隐藏
- 与剪贴板历史分开存储，互不干扰

**界面**

- Liquid Glass 风格：多层半透明玻璃、边缘高光、随背景光斑折射
- 按钮有悬停光泽扫过、按下内凹回弹的手感
- 右上角 **中 / EN** 一键切换整个界面语言，选择会记住
- 顶部胶囊分段控件，切换有弹簧缓动
- 尊重系统「减少动态效果」设置

**系统集成**

- 全局热键 `Alt + C`（被其他软件占用时不崩溃，只写日志）
- 托盘常驻，关闭窗口仅隐藏
- 单实例：重复启动不会开第二个，只把原窗口唤到前台
- **不写开机自启动项**（早期版本会强制写入，已移除）

## 数据存在哪

全部在本机 WebView 的 `localStorage` 里，不上传、不联网、无账号、无统计。

卸载程序或清除应用数据会一并删除历史记录，重要内容请自己另外留一份。

## 怎么运行

**方式一：直接跑编译好的 exe**

```
my-clipboard.exe
```

双击即可，或在项目根目录双击 `启动.bat`。

**方式二：自己编译**

需要 Node.js 18+、Rust 工具链、Visual Studio C++ 生成工具。

```powershell
npm ci
npm run build                                      # 构建前端产物
cd src-tauri
cargo build --release --features custom-protocol --target x86_64-pc-windows-msvc
```

产物在 `src-tauri\target\x86_64-pc-windows-msvc\release\my-clipboard.exe`，复制到根目录覆盖 `my-clipboard.exe` 即可。

> `--features custom-protocol` 不能省。少了它编译出来的程序会去连 `http://localhost:1420` 的开发服务器，双击后看到的是「无法访问此页面」。

**开发时热重载**

```powershell
npm run tauri dev
```

## 已知限制

- 只有 64 位 Windows 版本，没有跨设备同步
- 密码本没有做加密存储，敏感信息请谨慎使用
- 剪贴板历史记录条数没有上限，长期使用需要手动清理

## 来源

本项目基于 [hhrsc/clipboard](https://github.com/hhrsc/clipboard)（MIT）修改，在原版基础上重做了界面（Liquid Glass）、加入中英文切换、单实例保护，并移除了强制开机自启动。

原版的设计思路与基础功能归属原作者。

## License

[MIT](LICENSE)
