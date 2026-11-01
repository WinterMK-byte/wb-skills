---
name: ollama-gpu-diagnose
description: 诊断本地 Ollama 为何跑在 CPU 而非 GPU、为何吃系统内存而非显存。当用户说「ollama 占内存」「为什么不用显存」「ollama 好慢」「本地模型跑得慢」「GPU 没被用上」，或需要判断本地 LLM 推理是否在用显卡时使用。触发词：ollama、显存、GPU 推理、本地模型慢、qwen2.5、embedding 模型、内存占用。
---

# Ollama GPU / 内存诊断

## 一句话口径

**Ollama 占系统内存而不是显存 = 它根本没在用显卡，回退到了纯 CPU 推理。** CPU 推理只能吃 RAM，这是必然结果，不是"配置错了"。

判别只需一条命令：

```bash
ollama ps
```

看 **`PROCESSOR`** 列：
- `100% GPU` → 正常，吃显存
- `100% CPU` → **回退 CPU，吃内存**（核心症状）
- `xx%/yy%` → 混合，显存装不下全部

再看 `api/ps` 的 **`size_vram`** —— 这是最硬的判据：
- `size_vram: 0` → 显存占用 **0 字节**，铁证

## ★ 本地状态（2026-10-07 已修复）

**修复前**：独显 `Status = Error` → Ollama 探测 `total_vram="0 B"` → 两模型 100% CPU → 吃 7.37 GB 内存。
**修复后**：独显 `Status = OK` → CUDA 认出 5060，`total_vram="8.0 GiB"` → 模型 **100% GPU**，内存零占用。

| 检查项 | 修复前 | 修复后 |
|---|---|---|
| 独显 `Win32_VideoController.Status` | **`Error`** | **`OK`** |
| `nvidia-smi` | 权限不足 / 失败 | **正常输出表** |
| 驱动 | OEM 旧版 | **582.05** |
| Ollama 推理后端 | `library=cpu` | **`library=CUDA`** |
| `total_vram` | **`0 B`** | **`8.0 GiB`** |
| 模型运行位置 | 100% CPU | **100% GPU** |
| `qwen2.5:7b` 内存占用 | 5.22 GB | **0.00 GB** |
| 生成速度 | 5.18 tok/s | **63.2 tok/s（↑12×）** |

修法：设备管理器看独显黄叹号 → NVIDIA 官网重装驱动（RTX 50 系建议 570+）→ 重启。

**修复后基线数据**（供日后对比）：

| 项 | 值 |
|---|---|
| 主机 | LAPTOP-RLIQHP3A |
| 独显 | NVIDIA GeForce RTX 5060 Laptop GPU |
| 驱动 | 582.05（CIM 里显示 OEM 格式 `32.0.15.8205`） |
| 显存 | 总 8151 MiB / 已用 1393 MiB / 可用 6418 MiB |
| Ollama 版本 | 0.34.0（**0.40.0 可升级，非必需**） |
| 常驻模型（触发后） | `qwen2.5:7b` 占显存 4.64 GB，内存 0.00 GB |
| 生成速度 | **63.2 tok/s** |

## 一键体检脚本（现成的，别重写）

```
（诊断脚本按本文件正文的步骤自行拼装即可，无需外部文件）
```

双击即可，无需管理员。6 段检查：
1. 服务状态（**没跑会自动拉起**）
2. 显卡硬件 + 驱动 + **Status**（判据核心）
3. `nvidia-smi` 显存实况
4. 日志里的 GPU 探测记录
5. **★ 实测推理** —— 真的触发一次模型，读 `/api/ps` 看显存分配（最硬的判据）
6. 环境变量

自动出结论 + 留档 `_ollama_diag_<时间戳>.txt`。

**不要依赖 `nvidia-smi` 作为唯一判据** —— 沙箱 + 非管理员下会报 `insufficient permissions`。脚本以 `Get-CimInstance Win32_VideoController` 为主，`nvidia-smi` 只作补充。

**★ `Win32_VideoController.Status` 是判据核心**：`OK` = 正常；`Error` = 设备管理器带感叹号，Ollama 看不见它。

## 排查路径（按顺序，别跳步）

```
1. ollama ps 看 PROCESSOR
   ├─ 100% GPU → 正常，收工
   └─ 100% CPU → 继续
2. Win32_VideoController 查独显 Status
   ├─ Error / 查不到 → ★ 驱动问题（最常见）
   │   → 设备管理器看有无黄色感叹号
   │   → NVIDIA 官网下对应型号驱动重装（RTX 50 系建议 570+）
   │   → 重启后管理员 CMD 跑 nvidia-smi 确认出表
   └─ OK 但仍 CPU → Ollama/后端问题
       → 升级 Ollama（旧版对 Blackwell 支持不全）
       → 设 CUDA_VISIBLE_DEVICES=0 + OLLAMA_VULKAN=false 锁死 N 卡
3. 复查：ollama ps 的 PROCESSOR 应变 100% GPU
```

## 常见误判（踩过的坑）

- **★★「变慢了就以为回退 CPU」→ 先看「倍数」**（2026-10-09 补）：本地 **GPU 63 tok/s vs CPU 5 tok/s ≈ 12.6 倍**。
  所以**只慢 1~2 倍绝不是设备切换**，而是**GPU 时间片争用**（显示合成 / 其他进程抢 GPU）；
  **慢一个数量级（10 倍+）才是真的掉回了 CPU**。判据 = 「慢的倍数是不是数量级」，比 `ollama ps` 更快、且不用等模型加载。
  （实例：RAG 评测某次 `avg` 从 26s 变 46.9s = 1.8 倍 → 判定为争用，不是回退 CPU。）

- **❌「返回 200 就说明在用 GPU」** —— 不看 `size_vram` 就是瞎猜。
- **❌「有 NVIDIA 显卡就一定用上了」** —— 本地就是反例：显卡在、驱动在，但 **Status=Error**，Ollama 直接看不见。
- **❌「驱动版本号看着挺新」** —— `32.0.15.8205` 是 **OEM 换算格式**，末 5 位 `58205` → 582.05。要看 NVIDIA 官方格式。
- **❌「用 nvidia-smi 查就行」** —— 沙箱 + 非管理员都跑不了，得用 CIM。
- **❌「小模型放 CPU 也一样」** —— `qwen3-embedding:0.6b` 才 2.15GB，8GB 显存放它绰绰有余。
- **★★「Ollama 服务反复重启/不稳定」** —— **在 WorkBuddy 会话里跑 `ollama` 命令会自动拉起 `ollama app.exe`，而它跑在沙箱进程组里，命令一结束就被回收** → 表现为「服务出现又消失」。这不是 Ollama 的毛病，是沙箱铁律。**要判断服务是否真稳定，让用户手动启动后脱离会话观察**，或把「启动→就绪→触发→读结果」压在**同一条命令内**完成。

## 本地 RAG 影响面

| 组件 | 模型 | 影响 |
|---|---|---|
| 检索改写/重排 | `qwen2.5:7b`（本地 Ollama） | 现在 100% GPU，63 tok/s；修复前 5 tok/s，一次评测要跑几十分钟 |
| 向量化 | `qwen3-embedding:0.6b` | 现在上 GPU，建库/检索提速 |

代码位置：`kb_search_v4.py` 的 `llm_det()` / `llm_rerank()` 走 `k3.OLLAMA_CHAT`。

## 环境变量速查

| 变量 | 作用 | 本地值 |
|---|---|---|
| `OLLAMA_KEEP_ALIVE` | 模型驻留时长（越大越占显存/内存） | `30m` |
| `OLLAMA_KV_CACHE_TYPE` | KV 缓存量化 | `q8_0` |
| `OLLAMA_VULKAN` | Vulkan 后端（会抓 AMD 集显） | 未设（默认 true） |
| `OLLAMA_IGPU_ENABLE` | 是否允许用集显 | 未设（默认关闭） |
| `OLLAMA_NUM_GPU` | 卸载到 GPU 的层数 | 未设（自动） |
| `CUDA_VISIBLE_DEVICES` | 锁死用哪块 N 卡 | 未设 |

## 显存不够时的优化（8GB 的取舍）

1. **调短 `OLLAMA_KEEP_ALIVE`** → `5m`，模型更快换出
2. **减小上下文** → 7B 当前 8192，降到 4096 省 KV 缓存
3. **建库前先腾显存** → `ollama stop qwen2.5:7b`，把 8GB 全留给 embedding
4. **优先级**：embedding（2.15GB）优先保 GPU；7B（4.64GB）装不下时允许部分回退 CPU

## Windows 沙箱注意（踩过的）

- 会话里 `subprocess.Popen` 起的东西会随命令结束被回收（Job Object）→ 常驻服务必须让用户手动启动（Startup 目录 `.vbs` 或双击）。
- `HTTP_PROXY` 被沙箱设成 `127.0.0.1:<port>`，会拦截 `localhost` 请求返回 502 → Python 里要 `build_opener(ProxyHandler({}))` 显式禁用代理。
