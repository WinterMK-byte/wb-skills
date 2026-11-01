# wb-skills

六个每天真在用的 WorkBuddy / Claude Skills 兼容 skill。共同点是：**都是踩过坑之后才写下来的**，所以里面没有「建议考虑…」这种话，只有判据、实测数字、和「这么做过一次，翻车了」。

不是工具集，是排障手册。装不装随你，读一遍也不亏。

---

## 包含什么

| Skill | 它解决什么 | 里面有什么值得看的 |
|---|---|---|
| **windows-popup-diagnose** | Windows 上「一直弹窗 / 闪黑窗 / 终端窗口反复弹」，找不到元凶进程 | 用窗口事件钩子抓现行 + 回溯父进程链。含「`pythonw.exe` 计划任务本身不产生窗口，真凶是它 spawn 的子脚本里的 `git`/`cmd`/`schtasks`」这条反直觉结论 |
| **ollama-gpu-diagnose** | Ollama 跑在 CPU 不是 GPU，吃系统内存不吃显存 | 「别用 `nvidia-smi` 查占用」→ 用 `Get-CimInstance Win32_VideoController`；模型不在官方计费表时用量解析失败的表现 |
| **dedupe-hash-verify** | 「磁盘满了，哪些能删」—— 两份目录树到底是不是互为副本 | 只读核验（哈希 + 内容级比对），先出报告不动文件。核心反例：**只按目录体积判断会误删 313 MB 不可再生资料** |
| **remix-single-file-ui** | 把一个单文件 HTML 应用交给云端模型重做视觉，同时保证功能和数据零损坏 | 「抽数据 → 只给视觉层 → 回填 → 五道闸口自检」的流水线，和四版迭代的真实得分对照表（最瘦那版最丑，被否） |
| **xxt-notice-monitor** | 超星学习通「空间通知中心」的课程 / 作业通知，想自动进日程 | 纯 HTTP、零浏览器、零 SSO 跳转。反直觉结论：**不用折腾 headless 注入父域 cookie，把域改对即可** |
| **workspace-naming** | 新会话落进无名目录（`<时间戳>`），历史会话失联 | 为什么目录不能改名（路径写死在 db 里 240+ 处）；唯一根治办法是建任务时选工作空间 |

---

## 安装

这些是 Agent Skills 格式（`SKILL.md` + YAML frontmatter），WorkBuddy 与 Claude Code 都能读。

```bash
git clone https://github.com/WinterMK-byte/wb-skills.git

# WorkBuddy（用户级，所有项目可用）
cp -r wb-skills/skills/* ~/.workbuddy/skills/

# Claude Code（项目级）
mkdir -p .claude/skills && cp -r wb-skills/skills/* .claude/skills/
```

装完**重启客户端**。技能的清单在会话启动时固定，热加不会立刻生效。

> 如果你的客户端把 skill 标成「仅手动调用」，检查 `~/.workbuddy/settings.json` 的 `skillOverrides`：
> 键**不存在** = 模型可自动调用；`user-invocable-only` = 只能手打 `/名字`；`off` = 关闭。
> 想恢复自动调用就是删掉这个键。

---

## 用之前要改的占位符

脱敏时留下的占位符，按你的环境替换：

| 占位符 | 换成 |
|---|---|
| `<用户名>` | 你的 Windows 账户名 |
| `<产出目录>` | 你的 Agent 产出总目录 |
| `<uid>` / `<会话id>` | 你客户端数据目录里的实际值 |
| `<会议号>` / `<班级码>` | 你自己的 |
| `<时间戳>` | 客户端生成的目录名 |

`windows-popup-diagnose` 里提到的 `WpsUpdateTask_<用户名>`、`my-watch-daemon` 是**示例任务名**，请对照你自己 `Get-ScheduledTask` 的输出。

---

## 一条共同的设计取向

这六个 skill 都刻意写成「**给判断，不只给步骤**」：

- 每条结论都带**实测日期**，因为外部状态会变（Windows 更新、驱动版本、客户端行为）
- 关键处给**反例**：「这样判断会出错，具体错在哪」
- 有争议的地方**写清前提**，而不是给一个看起来确定的答案

原因很实在：步骤只能解决遇到过的那个问题，判断力才能解决下一个没遇到过的。

---

## License

MIT。随便用、随便改、商用也行。

---

## 相关

- [math-lecture-standard](https://github.com/WinterMK-byte/math-lecture-standard) — 数学讲解与出页标准（R1–R13 铁律 + 四步流水线 + 五关校验）
