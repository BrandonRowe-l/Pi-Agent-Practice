# 实践课1：Pi Agent 安装与使用

**环境信息**

- 系统：Windows（PowerShell）
- Pi 版本：1.5.0（官方安装器 `irm https://pi.dev/install.ps1 | iex`）
- 依赖：Node.js v24.18.0 / npm 11.16.0 / Git 2.55.0
- 练习目录：`E:\Pi Agent\pi-test`
- 模型配置：DeepSeek，API Key 方式（`/login` → Sign in with an API key），`pi auth check` 返回 ready

---

## 任务一 本地安装并启动 ✅

安装命令来自官方 README，安装后重新打开终端，在 pi-test 目录启动 `pi`。启动界面显示版本号 1.5.0、工作目录与底部模型状态栏。

截图：`assets/S1-install.png` — Pi 1.5.0 启动界面（版本、工作目录、deepseek-flash · low 状态栏可见）

## 任务二 配置模型并完成一次会话 ✅

通过 `/login` → **Sign in with an API key** 选择 DeepSeek 并提交 API Key，配置写入 `~/.pi/agent/auth.json`，`pi auth check --provider deepseek --model deepseek-flash` 返回 `ready`。发送 `hello` 收到模型回复，会话正常。

截图：`assets/S2-session.png` — hello 会话与模型回复

## 任务三 切换模型与思考级别 ✅

- 思考级别：Shift+Tab 打开 Thinking Level 选择器，**low（Light reasoning ~2k）→ high（Deep reasoning ~16k）**
- 模型：模型选择器中 **deepseek-flash → deepseek-v4-pro**
- 切换后发送短问题验证，会话正常

选择说明：简单问答用低思考级别省时间省用量；复杂推理/代码任务用 high + 更强模型，代价是等待时间和 token 消耗增加。

截图：`assets/S3-thinking-level.png`（思考级别选择器）、`assets/S4-model-switch.png`（切换后状态栏 deepseek-v4-pro · high）

## 任务四A 使用 @ 引用代码文件 ✅

在 pi-test 创建 `hello.py`（add 函数 + `print(add(2, 3))`），输入 `@` 选中该文件并发送解释请求。Pi 正确解释 add 函数功能并给出运行输出 `5`，未修改文件。

截图：`assets/S5-attach-reference.png`

## 任务四B 粘贴小图片并测试 ✅

自制 200×200 测试图片（蓝色方块 + 白色数字 7，`test-image.png`），在支持图片输入的模型下用 Ctrl+V 粘贴，出现附件提示后发送描述请求。Pi 正确识别：蓝色、正方形、数字 7。

截图：`assets/S6-image-test.png`

## 任务五 安装并使用一个插件 ✅

- 选型：[pi-web-access](https://pi.dev/packages)（pi.dev/packages，npm 包），作用是给 Pi 增加网页抓取工具，无需额外配置
- 安装：`pi install npm:pi-web-access`（用户级）
- 加载验证：退出后重新运行 `pi`，启动画面 `[Extensions]` 区域出现 `pi-web-access/dist`
- 功能试用：请求"请获取 https://example.com 的页面标题"，Pi 调用插件工具成功返回 `Example Domain`

截图：`assets/S7-extension.png` — [Extensions] 加载信息 + 一次实际调用结果

## 任务六 让 AI 安装一个 PPT Skill（选做）✅

- 从 https://github.com/anthropics/skills 下载 `skills/pptx` 完整目录到项目 `.pi/skills/pptx`，保留 SKILL.md、LICENSE.txt 与 scripts 全部支持文件
- 源 commit：`e4bb5d960d87126ff3b3a8af1a104b7db67560ba`（main，2026-11-16）
- 安装（sparse checkout 下载 → 复制到位）→ 重启发现（Pi 自动扫描项目 `.pi/skills`）→ 调用验证（询问名称/用途/目录，回复确认 `pptx`，路径指向本次安装位置）三层全部通过

截图：`assets/S8-skill-discovery.png`（重启后发现）、`assets/S9-skill-call.png`（调用与回复）

---

## 总结

| 任务 | 截图 | 状态 |
|------|------|------|
| 一 安装启动 | S1 | ✅ |
| 二 配置模型与会话 | S2 | ✅ |
| 三 切换模型/思考级别 | S3、S4 | ✅ |
| 四A @ 引用文件 | S5 | ✅ |
| 四B 粘贴图片 | S6 | ✅ |
| 五 插件安装与试用 | S7 | ✅ |
| 六 PPT Skill（选做） | S8、S9 | ✅ |

详细图文报告见 [01-Pi-Agent-作业报告.md](./01-Pi-Agent-作业报告.md)。
