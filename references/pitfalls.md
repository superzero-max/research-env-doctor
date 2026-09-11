# 真实踩坑清单

> **这份清单里的每一条都是真实发生过的**，不是网上抄的、不是理论上可能的。
> 排查新问题时，**优先假设是这些陷阱的变体**。

格式：现象 → 根因 → 怎么发现 → 怎么修 / 怎么避免

---

## 陷阱 1：PowerShell 变量名不区分大小写导致写错目标

**现象**：
脚本执行到"写入"那一步失败，报错信息里**目标路径变成了完全无关的东西**。

真实案例：给注册表写 PATH，报错显示
```
[错误] 写入失败: 属性 string Path=... 不存在或找不到。
回滚命令：Set-ItemProperty -Path 'e:\node\' ...
```
—— 目标变成了 `e:\node\`，而正确目标是 `HKLM:\SYSTEM\...\Environment`。

**根因**：
PowerShell 的**变量名不区分大小写**。脚本里：

```powershell
$Key = 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment'
...
foreach ($item in $items) {
    $key = $item.ToLowerInvariant()    # ← 这一行把上面的 $Key 覆盖了！
}
...
Set-ItemProperty -Path $Key ...        # ← 此时 $Key 已经是最后一个循环项的值
```

**怎么发现**：
看报错信息里的路径 —— 如果路径是"某个循环变量的最后一个值"，就是这个陷阱。

**怎么修 / 怎么避免**：
- 循环内的临时变量**换个明显不同的名字**：`$seenKey`、`$itemKey`、`$lookupKey`
- 写完脚本后**搜一遍**有没有同名不同大小写的变量：
  ```powershell
  Select-String -Path $script -Pattern '\$key\b'   # 小写
  Select-String -Path $script -Pattern '\$Key\b'   # 大写
  ```
- **写入前加校验**：确认目标存在、内容符合预期再写
  ```powershell
  if (-not (Test-Path $Key)) { throw "目标不存在: $Key" }
  ```

---

## 陷阱 2：YAML 的 `[]` 不能包块状列表

**现象**：
改了配置文件后，**服务直接起不来**。删掉文件里的 `[` 和 `]` 反而好了。

**根因**：
在 YAML 里，**`[` 开启的是 flow style（流式语法）**，里面只能写逗号分隔的内联项：

```yaml
# ✅ 合法：内联
[ {id: a}, {id: b} ]

# ❌ 致命：flow style 里写块状条目
[
    - id: foo
      name: bar
]
# → YAMLException: missed comma between flow collection entries
```

**关键认知**：模板里写 `[]` 表示**空数组**，那只在**文件为空时合法**。
**一旦要放条目，就必须去掉方括号、改用块状序列**：

```yaml
# ✅ 正确
- id: foo
  name: bar
```

**怎么发现**：
解析器会报 `missed comma between flow collection entries`，并指向 `[` 后面的第一行。

**怎么修 / 怎么避免**：
- **改完配置必须用真实解析器验证一遍**，不要靠肉眼判断
  ```python
  import yaml
  yaml.safe_load(open(path))     # 最简单
  ```
- 把"改完就验证"固化成习惯 —— **配置类文件永远不能凭感觉交付**

**教训延伸**：
这条踩坑导致过一个服务**完全无法启动**。配置文件的破坏性远大于脚本 bug ——
脚本错了只是跑失败，配置错了是**整个系统起不来**。

---

## 陷阱 3：`.bat` 里写中文导致窗口闪退

**现象**：
双击批处理文件，**窗口一闪就关**，什么都看不到。

**根因**：
cmd 默认按 **GBK** 解码批处理内容。如果文件用 **UTF-8** 保存且含中文，
中文会变成乱码，**乱码被当成命令名去执行**：
```
'系统' is not recognized as an internal or external command
```
每行都报错 → 脚本一路崩到结尾 → 窗口瞬间关闭，来不及看。

**怎么发现**：
```powershell
$b = [System.IO.File]::ReadAllBytes("script.bat")
"非 ASCII 字节数: $(($b | Where-Object { $_ -gt 127 }).Count)"
```
非 0 就是它。

**怎么修 / 怎么避免**：
- **批处理一律纯 ASCII**（只用英文），需要中文提示改用 `.ps1`
- 必须用 `.bat` 时，写成 ASCII 编码 + CRLF 换行：
  ```powershell
  [System.IO.File]::WriteAllText($p, $content, [System.Text.Encoding]::ASCII)
  ```
- 加 `pause` 兜底，避免出错就闪退

---

## 陷阱 4：整目录删除因单个文件被占用而整体失败

**现象**：
清理缓存目录时，`Remove-Item -Recurse` 报错：
```
The process cannot access the file 'xxx.nvph' because it is being used by another process.
```
**整个目录一个文件都没删掉。**

**根因**：
`Remove-Item -Recurse` 是**原子式**操作 —— 中途遇到一个锁定的文件就整个失败，
连已经删的不算数（或只删了一部分，结果不可预测）。

**怎么修 / 怎么避免**：
**逐文件删 + 统计成功失败**：
```powershell
$ok = 0; $locked = 0
Get-ChildItem $path -Recurse -File -Force -ErrorAction SilentlyContinue | ForEach-Object {
    try { Remove-Item $_.FullName -Force -ErrorAction Stop; $ok++ }
    catch { $locked++ }
}
"删除 $ok 个，跳过 $locked 个（被占用）"
```

**这条的价值**：遇到过 NVIDIA DXCache 15 GB，整目录删失败；
改成逐文件删后**成功清掉 15 GB，只剩 4 MB 被占用**。

---

## 陷阱 5：删文件要"先复制校验再删源"

**现象**（潜在风险）：
跨盘移动大目录，中途被中断（超时、断电、手动取消），
**不确定数据是丢了还是还在。**

**根因**：
`Move-Item` 跨卷时**不是原子操作**，是"复制 + 删除"两步。
中断时可能处于任意中间状态。

**怎么修 / 怎么避免**：
**永远用"复制 → 校验 → 才删源"的三步法**：
```powershell
# ① 复制（源不动）
robocopy $src $dst /E /COPY:DAT /R:1 /W:1
# ② 校验文件数 + 总字节数完全一致
# ③ 校验通过后才删源
```

**已验证的好处**：搬迁 56 GB 模型缓存时，先用"只复制"模式跑，
校验 173 个文件 / 60127224218 字节完全一致后才删源。
**即使中途出问题，数据也不会丢。**

**⚠️ 另一个发现**：跨盘 `robocopy /MOVE` 是"先复制后删源"，
所以**中断不会丢数据**（每个文件要么在源、要么在目标）。
但它**小文件极多时非常慢** —— 实测 29 万个文件走 USB，17 分钟只完成 13%。

---

## 陷阱 6：只看空间不看性能地选目标盘

**现象**（决策失误）：
把一个 56 GB 的模型缓存从系统盘搬到"空间很大的盘"，结果那个盘是**外置移动硬盘**。

**后果**：
- 读模型变慢
- **拔掉移动硬盘，目录联接就会失效**，依赖它的软件直接报错

**怎么避免**：
搬迁前必须同时确认**三件事**：
1. 空间够不够
2. **是内置盘还是外置盘**（决定性能与可靠性）
3. 被搬迁的东西**是否在被使用**（若在用，性能损失可接受吗）

**怎么查盘的类型**：
```powershell
Get-CimInstance Win32_DiskDrive | Select-Object Index, Model, InterfaceType
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" | Select-Object DeviceID, VolumeName
```
看 `Model` 里有没有 "USB"、"Basic"、品牌外置盘的字样。

---

## 陷阱 7：同一块物理盘的两个分区之间搬东西，不产生新空间

**现象**（认知误区）：
"C 盘满了 → 把东西搬到 Z 盘" —— 结果**总剩余空间没有任何变化**。

**根因**：
**C: 和 Z: 是同一块物理 SSD 的两个分区。**
在同一物理盘的分区之间移动文件，只是把空间从一个分区挪到另一个，**总容量不变**。

**怎么发现**：
```powershell
Get-CimInstance Win32_DiskDrive | ForEach-Object {
  $disk = $_
  Get-CimInstance Win32_DiskDriveToDiskPartition | Where-Object { $_.Antecedent.DeviceID -eq $disk.DeviceID } |
    ForEach-Object {
      $p = $_
      Get-CimInstance Win32_LogicalDiskToPartition | Where-Object { $_.Antecedent.DeviceID -eq $p.Dependent.DeviceID } |
        ForEach-Object { "{0} -> {1}" -f $disk.Model, $_.Dependent.DeviceID }
    }
}
```

**正确做法**：
要真正"腾空间"，必须搬到**另一块物理盘**上。

---

## 陷阱 8：`Get-Content -Encoding UTF8` 会吃掉 BOM

**现象**：
用 PowerShell 读一个带 BOM 的 UTF-8 文件、再用 `Set-Content` 写回，
**BOM 消失了**，导致后续按 BOM 判断编码的逻辑失效。

**怎么修 / 怎么避免**：
```powershell
# 读（-Raw 保留内容）
$content = Get-Content $path -Raw -Encoding UTF8

# 写回时显式指定是否要 BOM
# 需要 BOM（如 .ps1 含中文）：
[System.IO.File]::WriteAllText($path, $content, (New-Object System.Text.UTF8Encoding($true)))
# 不要 BOM（如 SKILL.md 的 frontmatter）：
[System.IO.File]::WriteAllText($path, $content, (New-Object System.Text.UTF8Encoding($false)))
```

**⚠️ 特别注意**：
**YAML/TOML/Markdown 的 frontmatter 文件绝对不能有 BOM** ——
BOM 会让 `---` 变成 `\ufeff---`，解析器认不出来，整个文件被静默丢弃。

---

## 陷阱 9：改完配置不验证就交付

**现象**：
改了配置文件，口头说"已改好"，**用户重启后系统起不来**。

**这条是整个清单里代价最大的**：破坏的不是一个功能，是**整个服务无法使用**。

**怎么避免（固化流程）**：
1. **改前备份**（带时间戳），并告诉用户备份在哪
2. **改后用真实解析器/真实启动验证**
3. **验证失败自动回滚**，不要留一个坏文件
4. **报告时说明验证结果**，而不是只说"已完成"

**验证示例（Python）**：
```python
import sys, yaml
try:
    data = yaml.safe_load(open(sys.argv[1]))
    print("OK", type(data).__name__, len(data) if hasattr(data, '__len__') else '')
except Exception as e:
    print("FAIL", e); sys.exit(1)
```

---

## 陷阱 10：注册表全盘递归搜索会超时

**现象**：
用 `Get-ChildItem HKLM:\SOFTWARE -Recurse` 找某个值，**跑几分钟不返回**。

**怎么避免**：
用**定点查询**代替全盘搜索：
```powershell
# 知道大概位置时：直接查
Get-ItemProperty 'HKLM:\SOFTWARE\Lenovo\LenovoAppStore' -ErrorAction SilentlyContinue

# 需要全盘找时：用 reg.exe（快得多）
reg query HKLM\SOFTWARE /f "要找的字符串" /s
```

---

## 附录：这些陷阱的共同模式

回头看，10 个陷阱可以归成 4 类：

| 类别 | 陷阱编号 | 共同点 |
|---|---|---|
| **编码 / 平台差异** | 3、8 | 文件"看起来对"，但解释方式不同 |
| **语言 / 工具的隐藏语义** | 1、2、4 | 用的是语法，踩的是语义 |
| **对环境的错误假设** | 6、7 | "空间大就行"、"搬过去就有空间" |
| **流程缺失** | 5、9 | 少了"校验"这一步 |
| **性能盲区** | 10 | 方法可行但代价不可接受 |

**最有价值的一条**：**所有"配置类"改动都必须验证（陷阱 9）** ——
因为它的破坏半径最大。
