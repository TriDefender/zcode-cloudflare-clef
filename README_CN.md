# zcode-cloudflare-clef

[English](./README.md)

[ZCode](https://zcode.z.ai) 插件：基于 [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/) **CLEF / CLEF Flash** 的结构化决策评测。不让聊天模型拍脑袋，而是由 MCP 工具 `clef_evaluate` 把具体状态和类型化问题（boolean / choice / score）发给一个小的校准模型，返回带概率分布的逐题答案——适合分诊、路由、严重度定级等"要校准置信度而不是要文采"的判断场景。

- MCP stdio server，零 npm 依赖（手写 JSON-RPC 分帧；PATH 上需有 Node ≥ 22）。
- 协议层（编码、响应校验、图片提取、上下文预算）逐字节移植自 [`openclaw-cloudflare-clef` 0.1.3](https://github.com/openclaw)（MIT）。
- 凭据 **DPAPI 加密落盘**（Windows）或经环境变量提供；永不写日志、永不明文写入配置文件。

## 安装（ZCode 桌面版）

1. **插件市场 → 添加 → 添加插件市场**，粘贴本仓库地址：`https://github.com/TriDefender/zcode-cloudflare-clef`
2. **个人 → cloudflare-clef → 安装**
3. 存储凭据（见下），完全重启 ZCode。

## 凭据

1. 准备 **Account ID**（32 位十六进制，Cloudflare 仪表盘 → Workers & Pages 概览）和 **API Token**（My Profile → API Tokens，权限选 **Workers AI → Run**）。
2. PowerShell 里运行一次（交互式输入，Token 不回显）：

   ```powershell
   powershell -ExecutionPolicy Bypass -File "<插件目录>\server\store-secret.ps1"
   ```

   两个值以 DPAPI 密文存于 `%USERPROFILE%\.zcode\secrets\cloudflare-clef.json`，绑定当前 Windows 用户——磁盘无明文。重跑即轮换；删除文件即吊销。
3. 非 Windows / 备选：设置用户级环境变量 `CLOUDFLARE_ACCOUNT_ID` 与 `CLOUDFLARE_API_TOKEN`；真实环境变量优先于加密存储。

如实说明安全边界：DPAPI 防的是文件外泄（备份、网盘同步、磁盘被拷），防不了同 Windows 用户下的其他进程。

## 用法

让模型调用 `clef_evaluate`。问题是"id → 类型化问题"的映射，id 任意（支持中文、斜杠）：

```json
{
  "model": "clef-flash",
  "state": "货物尚未电放，ETA 明天，需要联系起运港。",
  "questions": {
    "是否紧急": { "type": "boolean", "instructions": "Is this urgent?" },
    "route": { "type": "choice", "criteria": { "关务": "Customs clearance", "海运": "Ocean freight" } },
    "severity": { "type": "score", "criteria": ["None", "Moderate", "Critical"] }
  }
}
```

返回 `{model, answers, usage}`，答案以原始 id 为键。`model` 省略时用 `clef-flash`（更快更便宜）；`clef` 更强。图片支持内嵌 data URL、`{content_type,base64}` 对象或**绝对路径本地文件**（`.png/.jpg/.jpeg/.webp`，由 server 读盘嵌入）——聊天模型无法可靠生成大段 base64，真实截图请优先用路径。仅按输入计价（2026-10-20）：clef-flash 3,455 Neurons / M 输入 tokens，clef 21,818 / M；Workers Free 每天（UTC）免费 10,000 Neurons。

## 范围与副作用

- 网络访问：仅 POST `https://api.cloudflare.com/client/v4/accounts/<accountId>/ai/run/@cf/cloudflare/<model>`。无重定向、无遥测、无重试。
- MCP 启动时读取两个来自环境的凭据；评测时读取 state 中以绝对路径引用的本地图片文件（仅 png/jpg/jpeg/webp，单张 ≤ 4 MiB）；进程之外不写任何文件。
- 每次请求证据上限约 24k 文本字节与 4 张图片（PNG/JPEG/WebP，单张 4 MiB / 合计 8 MiB / 16 MP）；响应体读取上限 1 MiB。

## 开发

```shell
node --test cloudflare-clef/server/protocol.test.mjs   # 15 个协议测试
```

## 许可

MIT — 见 [LICENSE](./LICENSE)。协议层衍生自 MIT 许可的 OpenClaw 插件 `openclaw-cloudflare-clef` 0.1.3。
