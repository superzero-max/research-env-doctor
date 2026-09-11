# 常见报错对照表

> 用法：按症状找到对应段落 → 看「为什么」→ 执行「怎么修」。
> **每一条都包含原理**，目的是让你下次能自己判断，而不是背命令。

---

# 一、conda 相关

## 1.1 `conda: command not found`（明明装过）

**为什么**：
`conda` 不是一个独立可执行文件，而是靠 shell 初始化脚本注入的函数/别名。
如果 `~/.bashrc`（或 `.zshrc`）里没有那段 `conda init` 写入的内容，shell 就不知道 conda 的存在。
**在新开的 shell、非交互式 shell、或 `srun`/`sbatch` 作业里，这个问题特别常见** —— 因为作业脚本不加载交互式 shell 的配置。

**怎么查**：
```bash
grep -n "conda" ~/.bashrc              # 有没有初始化段
echo $PATH | tr ':' '\n' | grep conda  # PATH 里有没有 conda
ls ~/miniconda3/bin/conda ~/anaconda3/bin/conda 2>/dev/null   # 本体在哪
```

**怎么修**：
```bash
# 方式 1：执行初始化（推荐，一次性）
~/miniconda3/bin/conda init bash        # 换成你的实际路径
source ~/.bashrc

# 方式 2：临时用绝对路径
~/miniconda3/bin/conda activate myenv

# 方式 3（集群作业脚本里最可靠）：显式 source
source ~/miniconda3/etc/profile.d/conda.sh
conda activate myenv
```

**⚠️ 集群注意**：`sbatch` 提交的作业默认是**非交互 shell**，不读 `.bashrc` 的交互段。
**必须在作业脚本里显式 `source .../conda.sh`**，否则永远 `command not found`。

---

## 1.2 `CommandNotFoundError: Your shell has not been properly configured to use 'conda activate'`

**为什么**：
你能运行 `conda`（说明 PATH 有），但 `conda activate` 这个子命令需要 shell 函数支持。
旧版本 conda 或者被 `source activate` 的旧写法影响时会这样。

**怎么修**：
```bash
# 临时解决（当前会话）
eval "$(conda shell.bash hook)"
conda activate myenv

# 永久解决
conda init bash
exec bash      # 重开 shell
```

---

## 1.3 `Could not find conda environment: xxx`（环境明明存在）

**为什么**（三种可能，按概率排序）：
1. **你在错误的 conda 里找** —— 机器上有多个 conda 安装，`conda env list` 显示的不是同一个
2. **环境在非默认路径** —— 有些环境建在 `--prefix` 指定的位置，不在 `envs/` 下
3. **环境被删了一半** —— 目录还在但 `conda-meta` 缺失

**怎么查**：
```bash
conda info | grep -E "base environment|envs directories"
conda env list                       # 看完整列表，包括 --prefix 建的
ls -d <conda_root>/envs/*/conda-meta   # 哪些环境是完整的
```

**怎么修**：
- 情况 1 → 统一到一套 conda，或用绝对路径 `conda activate /full/path/to/env`
- 情况 2 → 直接用绝对路径激活
- 情况 3 → 环境已损坏，用 `conda env export` 的记录重建

---

## 1.4 conda 用久了特别慢 / 占空间巨大

**为什么**：
conda 会把下载过的每个包**永久缓存在 `pkgs/`** 目录，几十个包动辄 10+ GB。
而且每次操作都要扫描这个目录，越大多越慢。

**怎么查**：
```bash
du -sh $(conda info --base)/pkgs
du -sh ~/.conda/pkgs 2>/dev/null
```

**怎么修**：
```bash
conda clean -a -y     # 清包缓存 + 索引缓存 + tarball
```
**这个操作是安全的** —— 只删缓存，不动任何已安装环境。删掉的包下次需要时会重新下载。

---

# 二、Python 解释器相关

## 2.1 `python` 跑出来的版本不是我要的

**为什么**（按概率）：
1. **PATH 顺序问题** —— 系统里装了多个 Python，`PATH` 靠前的那个赢了
2. **旧的系统 Python 排在前面** —— 常见于预装 Python 的发行版
3. **conda 环境没激活** —— 你以为激活了，其实没有

**怎么查**（关键：看真实路径，不是版本号）：
```bash
which -a python python3        # 所有匹配项，按 PATH 顺序
python -c "import sys; print(sys.executable)"   # 真正在执行的那个
echo $PATH | tr ':' '\n'                        # PATH 顺序
```

**Windows**：
```powershell
where.exe python                       # 所有匹配
Get-Command python | Select-Object Source
$env:PATH -split ';' | Where-Object { $_ -match 'Python|anaconda|conda' }
```

**怎么修**：
```bash
# 方案 A（推荐）：用环境隔离，不依赖 PATH
conda activate myenv     # 或 source venv/bin/activate
which python             # 确认

# 方案 B：调整 PATH 顺序（改 ~/.bashrc 或系统环境变量）
export PATH="/your/preferred/python/bin:$PATH"

# 方案 C：直接调用绝对路径（脚本里最可靠）
/your/preferred/python/bin/python script.py
```

**⚠️ Windows 特有陷阱**：
Windows 的 `PATH` 是**先系统级、后用户级**。
所以如果**系统级**放着旧 Python（比如 Python 3.5），**用户级**的 Python 3.12 永远排在后面，
`python` 会被旧版本劫持 —— **必须把旧版本从系统级 PATH 里移除**，光调整用户级没用。

---

## 2.2 `ModuleNotFoundError: No module named 'xxx'`（明明 pip install 过）

**为什么**：
`pip` 和 `python` **不是同一个环境**。
最常见的场景：`pip` 指向环境 A，`python` 指向环境 B —— 包装到了 A，你跑的是 B。

**怎么查**（这一步能解决 90% 的这类问题）：
```bash
which python pip
python -m pip --version
python -c "import sys; print(sys.executable)"
# 对比：pip 的路径前缀 与 python 的路径前缀是否一致
```

**怎么修**：
```bash
# 黄金法则：永远用 python -m pip，而不是裸 pip
python -m pip install xxx

# 或先激活环境再装
conda activate myenv
python -m pip install xxx
```

**为什么 `python -m pip` 更可靠**：它强制用"当前 python 自己的 pip"，
杜绝了 pip 与 python 指向不同环境的问题。

---

# 三、共享库相关

## 3.1 `ImportError: libXXX.so: cannot open shared object file: No such file or directory`

**为什么**：
编译过的 Python 包（numpy、scipy、MDAnalysis、torch 等）在运行时需要**动态链接**外部 C/C++ 库。
`libXXX.so` 找不到，说明该库**没装**，或者**装了但不在动态链接器的搜索路径里**。

**怎么查**：
```bash
# ① 到底缺哪个库
ldd $(python -c "import sys; print(sys.executable)") | grep "not found"
# 对具体扩展模块也可以：
ldd $(python -c "import numpy; print(numpy.__file__)") | grep "not found"

# ② 这个库到底装在哪
find / -name "libXXX.so*" 2>/dev/null
```

**怎么修**（按推荐顺序）：

```bash
# 方案 A：用 conda 装（最省事，conda 会连依赖一起装）
conda install -c conda-forge <package>

# 方案 B：装系统库（Debian/Ubuntu）
sudo apt-get install lib<name>-dev

# 方案 C：手动加搜索路径（临时）
export LD_LIBRARY_PATH=/path/to/lib:$LD_LIBRARY_PATH
```

**⚠️ `LD_LIBRARY_PATH` 的坑**：
- 它是**临时**的，新开 shell 就没了
- 滥用它会造成**更隐蔽的冲突**（加载到了错误版本的库）
- **优先用方案 A/B**，把 C 当最后手段
- 要永久生效应写进 `.bashrc`，但**排查阶段先别写**，确认有效再说

**⚠️ 集群注意**：不要用 `LD_LIBRARY_PATH` 覆盖系统的 MPI 库，
这会导致 MPI 混用崩溃（见 MPI 段）。

---

## 3.2 `undefined symbol: xxx`（库找到了但符号缺失）

**为什么**：
库找到并加载了，但**版本不对** —— 里面没有你要的符号。
典型场景：软件 A 需要 `libz.so.1.2.11` 的某个符号，但加载到的是更旧/更新的 `libz.so.1.2.8`。

**怎么查**：
```bash
# 确认实际加载的是哪个
LD_DEBUG=libs python -c "import 你的模块" 2>&1 | grep -i "libXXX"
```

**怎么修**：
- 找出**谁引入了错误版本**，统一版本
- conda 环境内的问题，优先 `conda install` 而不是 pip install（conda 会统一依赖版本）

---

# 四、MPI 相关（MD 高发）

## 4.1 `mpirun` 报错 / 程序立即崩溃 / 提示 MPI 版本不匹配

**为什么**：
MPI 有多个互不兼容的实现：**OpenMPI、MPICH、Intel MPI、MVAPICH**。
GROMACS/LAMMPS 在**编译时**链接了某一个。运行时如果 `mpirun` 来自**另一个实现**，
就会崩溃或报奇怪的错。

**怎么查**（关键命令）：
```bash
which mpirun mpicc
mpirun --version                       # 是哪个 MPI
ldd $(which gmx) | grep -i mpi         # 软件实际链接哪个
```

**判断**：上面两条显示的 MPI **必须属于同一实现**。

**怎么修**：
```bash
# ① 清理环境里的 MPI 相关变量
env | grep -E "MPI|OMPI|MPICH|I_MPI"
unset LD_LIBRARY_PATH    # 排查阶段先清空（确认没有别的东西依赖它）
unset PATH_TO_MPI...

# ② 用 module 系统切换到匹配的实现（集群上最规范）
module avail
module load openmpi/4.x.x            # 加载与该软件编译时一致的版本
module load gromacs/2024.x

# ③ 或者用软件自带的 mpirun（如果发行版打包了）
```

**⚠️ 集群常见错误**：在作业脚本里 `module load` 了 A 版本的 MPI，
但 `sbatch` 环境的 `LD_LIBRARY_PATH` 里还残留着 B 版本 → 崩溃。
**排查时先 `echo $LD_LIBRARY_PATH` 看有没有脏东西。**

---

## 4.2 `There are not enough slots available in the system`

**为什么**：
`mpirun -np N` 的 N 超过了你**实际分到的核数**（在集群上，超了会被拒绝）。

**怎么修**：
```bash
# 在集群上：让 MPI 用调度器分配的核，而不是自己指定
srun gmx mdrun ...            # Slurm：用 srun，不用 mpirun
# 或
mpirun -np $SLURM_NTASKS gmx mdrun ...

# 本机：确认核数后调整
nproc
mpirun -np $(nproc) --oversubscribe gmx mdrun ...   # 允许超配（会变慢）
```

---

# 五、集群 / 作业提交相关

## 5.1 作业提交后立刻失败（`FAILED` 或 `CANCELLED`，几乎没运行）

**为什么**（按概率）：
1. **脚本没有可执行权限**
2. **shebang 行错误**（`#!/bin/bash` 写错或文件是 Windows 换行 CRLF）
3. **walltime 超限** —— 请求时间超过分区限制
4. **资源请求不合理** —— 核数/内存超限
5. **队列/账户问题** —— 余额不足、账户错误

**怎么查**：
```bash
ls -l job.sh                            # 有没有 x 权限
head -1 job.sh | cat -A                 # 看 shebang 行末尾有没有 ^M（CRLF 的标志）
sacct -j <jobid> --format=JobID,State,ExitCode,DerivedExitCode
cat slurm-<jobid>.out                   # 看真实报错
```

**怎么修**：
```bash
chmod +x job.sh
# 如果是 CRLF 问题（Windows 编辑后上传的脚本）：
sed -i 's/\r$//' job.sh          # 去掉所有 CR
# 或
dos2unix job.sh
```

**⚠️ 这是最容易忽略的坑**：**在 Windows 上编辑的脚本上传到 Linux 集群，换行符是 CRLF，
Linux 会把 `\r` 当成命令的一部分** → 报 `command not found` 或各种诡异错误。
**永远用 `dos2unix` 或 `sed -i 's/\r$//'` 处理一遍。**

---

## 5.2 作业跑了一会儿被杀（`OUT_OF_MEMORY` / `TIMEOUT`）

**为什么**：
- `OUT_OF_MEMORY`：请求的内存不够（MD 的内存需求随体系大小增长很快）
- `TIMEOUT`：超过 walltime

**怎么修**：
```bash
# 看实际用了多少
sacct -j <jobid> --format=JobID,MaxRSS,MaxVMSize,Elapsed,Timelimit

# 调整请求（Slurm 示例）
#SBATCH --mem=64G          # 加大内存
#SBATCH --time=48:00:00    # 加长时间
```

**预防**：先用**小体系短时间**试跑一次，量出真实资源需求，再提交大作业。

---

# 六、权限相关

## 6.1 `Permission denied`（文件明明是我的）

**为什么**：
1. 缺少**执行权限**（下载的脚本常见）
2. 目录**上级**没有进入权限
3. 文件在**只读挂载**上（集群的共享目录常是只读的）
4. Windows 上文件被进程占用

**怎么查**：
```bash
ls -ld . .. <file>
id
mount | grep $(df --output=target . | tail -1)   # 是否只读挂载
```

**怎么修**：
```bash
chmod +x script.sh
# 临时提权（不推荐长期）
sudo chown $USER <file>
```

**Windows**：文件被占用时，先关掉占用它的程序（用任务管理器或 `handle.exe` 查）。

---

# 七、Windows 特有

## 7.1 `.bat` / `.cmd` 脚本窗口一闪就关

**为什么**：
里面写了**中文**。cmd 默认按 **GBK** 解码，而文件存成 **UTF-8** →
中文变成乱码 → 乱码被当成命令去执行 → 每行都报 `'xxx' is not recognized` → 脚本崩到底、窗口瞬间关闭。

**怎么查**：
```powershell
$b = [System.IO.File]::ReadAllBytes("script.bat")
"非 ASCII 字节数: $(($b | Where-Object { $_ -gt 127 }).Count)"
```
**非 0 = 有中文/特殊字符**，就是这个问题。

**怎么修**：
- **把批处理改成纯 ASCII（英文）**
- 需要中文提示时，改用 PowerShell 脚本（`.ps1` 能正确处理 UTF-8）

---

## 7.2 PowerShell 脚本报奇怪的语法错误（中文注释导致）

**为什么**：
PowerShell 5.1 读 `.ps1` 时，**如果没有 BOM**，会按 **ANSI(GBK)** 解码 →
中文注释变乱码 → 可能吃掉引号、破坏语法 → 报 `Unexpected token`。

**怎么查**：
```powershell
$b = [System.IO.File]::ReadAllBytes("script.ps1")[0..2]
if ($b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF) { "有 BOM ✓" } else { "无 BOM ✗ 这就是问题" }
```

**怎么修**：
```powershell
$content = Get-Content $path -Raw -Encoding UTF8
[System.IO.File]::WriteAllText($path, $content, (New-Object System.Text.UTF8Encoding($true)))   # $true = 写 BOM
```

**⚠️ 注意**：`Get-Content -Encoding UTF8` 会**吃掉** BOM，所以写回时必须显式指定 `$true`。

---

## 7.3 磁盘空间不够 / 缓存占满

**常见大块头**（按优先级清理）：
| 位置 | 说明 |
|---|---|
| `%LOCALAPPDATA%\NVIDIA\DXCache`、`GLCache` | 显卡着色器缓存，**删了自动重建** |
| conda `pkgs/` | 包缓存，`conda clean -a -y` |
| `%LOCALAPPDATA%\Temp`、`C:\Windows\Temp` | 临时文件 |
| `C:\Windows\SoftwareDistribution\Download` | Windows 更新残留 |
| site-packages 里 `~` 开头的目录 | **pip 卸载残留**，纯垃圾 |

**⚠️ 绝不要碰**：
- `C:\Windows\Installer`（删了软件无法卸载/修复）
- `C:\Windows\WinSxS`

**⚠️ 删缓存目录要逐文件删**：
整目录 `Remove-Item -Recurse` 会因为**一个被占用的文件**而整体失败。
```powershell
Get-ChildItem $p -Recurse -File -Force | ForEach-Object {
  try { Remove-Item $_.FullName -Force -ErrorAction Stop } catch { $locked++ }
}
```
