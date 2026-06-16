# CC Switch 打开中文路径子文件夹报 `'/d' 不是内部或外部命令`

> **CC Switch 3.16.3 · Windows · 已确认的 bug · 附 workaround**

---

## 症状

用 CC Switch 打开 `D:\某中文文件夹\子目录` 这类**中文名路径**时，终端闪现并报：

```
'/d' 不是内部或外部命令，也不是可运行的程序或批处理文件。
```

而打开纯英文路径（如 `D:\projects\my-app`）完全正常。

---

## 根因

**CC Switch 3.16.3 生成的启动 `.bat` 换行符不一致，导致 cmd.exe 在中文多字节路径下解析错位。**

具体来说：

### 1. 源码层面

CC Switch 的 `src-tauri/src/commands/misc.rs` 中，`launch_windows_terminal` 函数（约 L3054）用 Rust 的 `format!` 宏拼接 bat 内容：

```rust
let content = format!(
    "@echo off
{cwd_command}
echo Using provider-specific claude config:
...
claude --settings \"{}\"
...");
```

模板里的换行是**裸 `\n`（LF）**，但 `build_windows_cwd_command_str`（约 L3136）生成的 cwd 行用了 `\r\n`（CRLF）：

```rust
format!("cd /d \"{escaped}\" || exit /b 1\r\n")
```

结果：整个 bat 文件**混合换行**——首行 `@echo off` 后是孤立 `0A`（LF），cwd 行后是 `0D 0A`（CRLF）。

### 2. 为什么会炸

cmd.exe 按行解析 bat 时，遇到混合换行 + 中文多字节字符（GBK 编码下每个中文字符占 2 字节），行边界计算错位。`cd /d "D:\某中文路径"` 被错误地从 `/d` 处截断，`/d` 变成独立的命令 token → 报 `'/d' 不是内部或外部命令`。

### 3. 关键复现矩阵

| bat 换行 | 路径类型 | 结果 |
|----------|---------|------|
| 混合（LF + CRLF）| 中文路径 | **必报错** |
| 混合（LF + CRLF）| 英文路径 | 正常（ASCII 单字节，侥幸不错位）|
| 全 CRLF | 中文路径 | 正常 |
| 全 CRLF | 英文路径 | 正常 |

- 与代码页无关（936 GBK 和 65001 UTF-8 都复现）
- 与终端类型无关（cmd / Windows Terminal / PowerShell 都复现）
- 唯一变量：**混合换行 + 多字节路径**

---

## 修复方案

### 方案一：英文 Junction（推荐，最稳）

用 Windows 的 NTFS Junction 给中文路径创建一个英文别名：

```powershell
# 以管理员或普通用户权限均可（junction 不需要管理员）
New-Item -ItemType Junction -Path "D:\projects\my-chinese-folder" -Target "D:\某中文文件夹"
```

然后在 CC Switch 里用英文路径 `D:\projects\my-chinese-folder` 打开项目。

**优点**：
- 对所有程序透明（读写 = 直接访问原目录）
- 不占额外空间
- 扛得住 CC Switch 更新（不依赖 bat 内容）
- 零维护成本

**验证**：即使 CC Switch 的混合换行 bat，cwd 指向英文 junction 时 `cd /d` 完全正常。

#### CC Switch 操作步骤

CC Switch 没有"编辑路径"功能——每次打开终端都是弹 Windows 原生文件夹选择器。操作步骤：

1. 点击 provider 卡片上的"打开终端"按钮
2. 弹出 Windows 文件夹选择对话框
3. **导航到 `D:\AI_code\`**，在列表中找到 `cuttlefish-ppt`（与中文名并列）
4. 点选它 → 确认

Junction 在文件夹选择器中显示为普通文件夹，选中后返回的路径是 junction 路径（不会被解析为中文目标路径），因此 bat 文件中的 `cd /d` 始终使用英文路径。

> **注意**：不需要每次重新导航——Windows 文件夹选择器会记住上次访问的位置。

### 方案二：bat 监控自动修复（备选）

写一个后台脚本，监控 `%TEMP%\cc_switch_claude_*.bat`，一生成就立刻把所有 `0A` 替换成 `0D 0A`（统一 CRLF）。

```powershell
# 示例：监控并修复（需提前启动）
$filter = "cc_switch_claude_*.bat"
$watcher = New-Object System.IO.FileSystemWatcher
$watcher.Path = [System.IO.Path]::GetTempPath()
$watcher.Filter = $filter
$watcher.EnableRaisingEvents = $true

Register-ObjectEvent $watcher Created -Action {
    Start-Sleep -Milliseconds 50  # 等写完
    $content = [System.IO.File]::ReadAllBytes($event.SourceEventArgs.FullPath)
    # 替换裸 LF 为 CRLF
    $fixed = [byte[]]@()
    for ($i = 0; $i -lt $content.Length; $i++) {
        if ($content[$i] -eq 0x0A -and ($i -eq 0 -or $content[$i-1] -ne 0x0D)) {
            $fixed += 0x0D, 0x0A
        } else {
            $fixed += $content[$i]
        }
    }
    [System.IO.File]::WriteAllBytes($event.SourceEventArgs.FullPath, $fixed)
}
```

**缺点**：有竞态风险（bat 生成后毫秒级就执行），不保证 100% 修复。

### 方案三：上游修复（根治）

给 CC Switch 提 issue 或 PR，把 `misc.rs` L3066 的 `format!` 模板里裸 `\n` 全改成 `\r\n`：

```rust
// 修复前
let content = format!("@echo off\n{cwd_command}\n...");

// 修复后
let content = format!("@echo off\r\n{cwd_command}\r\n...");
```

重新编译即可根治。

---

## 排障速查

遇到 `'/d' ...` 报错时，按以下顺序排查：

| 检查项 | 说明 |
|--------|------|
| 错误主体是 `/d` 还是 `claude`？ | `/d` = 本 bug（换行符问题）；`'claude' 不是...` = PATH 问题（另一个 bug）|
| 路径是否含中文/特殊字符？ | 含 → 大概率触发本 bug |
| 换英文路径能否正常？ | 能 → 坐实中文路径触发 |
| CC Switch 版本？ | 3.16.3 确认有此 bug，后续版本待验证 |

---

## 环境

- CC Switch 3.16.3
- Windows 11（代码页 936 / 65001 均复现）
- cmd.exe / Windows Terminal / PowerShell 均复现

---

## 相关 Issue

- [CC Switch 打开 Claude Code 不是完全访问模式](https://github.com/xzyj50609/cc-switch-bypass-permissions)（另一个独立 bug，PATH 层面）

---

## License

Public Domain / MIT — 随意使用。
