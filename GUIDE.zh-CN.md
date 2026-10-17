# CUT&Tag 分析流程搭建与实践指南

版本 1.0.0 · 2026-10-17 · 面向使用者与执行 AI

## 1. 目标、实践依据与阅读方法

本指南帮助读者根据自己的电脑、数据和实验设计，搭建一套可记录、可检查、可恢复的 bulk 双端 CUT&Tag 分析流程。作者已在 Apple Silicon / macOS 上完成环境安装、真实人源数据分析、报告生成、结果核对和多轮排故；以下方法从这些实践中提炼。

本目录只提供文档和填写模板。通用命令可以作为实施起点；需要自定义程序的部分给出输入、输出、算法和检查要求，由读者或 AI 编写。所有示例路径均属于读者将创建的工作区，不需要作者电脑上的文件。

### 1.1 实践依据如何解释

| 内容 | 本机实践依据 | 本指南的边界 |
|---|---|---|
| 人源基础流程 | 真实数据完成 QC、修剪、竞争比对、重复标记、成对过滤、MACS3、片段轨道、TSS/基因体、注释、GO、known motif、MultiQC 和中文报告 | 不能据此保证其他数据或设计合格 |
| 下游扩展 | 利用保存的真实结果完成 KEGG、Excel、IGV 和报告展示检查 | 展示重建与下游补算分别记录，不称作全量重跑 |
| 加测、校准、恢复和重复性机制 | 有合成输入、组件和部分集成验证 | 不能替代真实同库关系、校准实验和独立重复的验证 |
| 平台 | 原生 osx-arm64；建设机器 Apple M4 Pro、24 GiB 内存 | 该配置是实践背景，不是最低硬件要求；Linux/Intel Mac/WSL2 尚需读者验收 |
| 公开示例 | 本轮核对方法、命令语法和资料结构 | 未把正文组合为新公开程序并完成从零运行 |

目前不把单细胞、单端、组间差异峰、de novo motif、真实 SEACR、IDR 或其他物种分析列为本文已验证的正式流程。工具装好了，也不能因此把这些项目标为已完成。

### 1.2 怎样执行

按“盘点 → 部署 → 小型验证 → 真实输入/参考核验 → 分阶段计算 → 数值/报告验收 → 归档”推进。复制 [运行记录](templates/RUN_RECORD.md) 和 [验收清单](templates/ACCEPTANCE.md)，逐项填写“通过 / 不通过 / 不适用 / 待确认”，附实际证据。空框、命令示例和 AI 的推测均不是通过记录。

正文中的“实现合同”是需要读者或 AI 落地的处理要求；没有附带同名私有脚本。遇到阻断项，保留日志并停止受影响下游，继续独立工作。

## 2. 先确认平台、输入与资源

### 2.1 平台选择

| 平台 | 部署方向 | 要验证的差异 |
|---|---|---|
| Apple Silicon Mac | 原生 osx-arm64，避免混入 osx-64 | 包架构、Java、R/Bioconductor、截图和文件权限 |
| Intel Mac | 原生 osx-64，重新求解和导锁 | 不能安装 ARM 构建；重新做工具链与真实数据检查 |
| Linux x86_64 / aarch64 | 匹配系统架构的 Miniforge 与 Bioconda | 包可用性、系统库、进程管理、无桌面 IGV 截图 |
| Windows | 在 WSL2 的 Linux 终端评估部署 | 按 Linux 架构安装，核对 Linux 文件系统、资源限额和图形支持 |

Bioconda 支持的平台见 [官方说明](https://bioconda.github.io/)。Windows 的 WSL2 建设见 [微软说明](https://learn.microsoft.com/en-us/windows/wsl/install)。这些来源说明平台能力，不证明本指南已在相应平台通过。

### 2.2 输入与资源盘点

开始前记录系统/架构、CPU、内存、可用空间、输入压缩体积、计划样本/文库数、参考物种/版本和期望模块。需要终端、文件读写及下载资源的能力；根据环境安排网络、安装权限与长任务执行。

容量预算同时覆盖：压缩 FASTQ、clean FASTQ、竞争 BAM、重复标记/名称排序 BAM、最终 BAM、排序临时文件、矩阵、图件、报告和一份必要归档副本。不要用“压缩输入大小 × 固定倍数”承诺峰值。以小批实测修订预算；线程、Java heap 和 samtools 每线程内存合并考虑，避免多个工具同时吃满内存。

活动输入优先复制到可靠的本地工作盘并核验。外盘读取成功一次不证明长期稳定；WSL 路线应核对输入和计算目录的真实挂载、权限、I/O 行为及可用空间。长任务接电并安排防睡眠，不能把关掉网页等同于停止计算。

## 3. 建立工作目录与运行记录

下面是新建工作区的 Bash 示例。先确认路径不属于已有部署；后续安装/输出路径都由 `CT_ROOT` 派生。不要移动已经创建的 Conda 前缀后默认它仍可运行。

```bash
export CT_ROOT="$HOME/cuttag_workspace"
mkdir -p "$CT_ROOT"/{00_docs,01_software/{config,locks,scripts},02_input,03_references,04_intermediate,05_results,06_records,07_cache/{downloads,conda_pkgs,tmp,plotting},envs}
export CONDA_PKGS_DIRS="$CT_ROOT/07_cache/conda_pkgs"
export CONDA_ENVS_PATH="$CT_ROOT/envs"
export TMPDIR="$CT_ROOT/07_cache/tmp"
export MPLCONFIGDIR="$CT_ROOT/07_cache/plotting"
export PYTHONNOUSERSITE=1
export OPENBLAS_NUM_THREADS=2
export OMP_NUM_THREADS=2
```

目录使用英文、数字、下划线或短横线，尽量避开空格、逗号和工具保留字符。中文用于说明。把这些变量保存在自己的运行入口中；每个新终端先读取并核对，不依赖上一次对话的临时状态。

| 位置 | 内容 |
|---|---|
| `02_input/<project>/` | 原始压缩 FASTQ、来源校验和接收清单；分析只读 |
| `03_references/<species>/<version>/` | FASTA/GTF、索引、区域、来源/校验、TxDb 和数据库快照 |
| `04_intermediate/<project>/<run>/` | 本次计算的 clean FASTQ、过程 BAM、片段与矩阵 |
| `05_results/<project>/<run>/` | 冻结的正式结果和完整报告包 |
| `06_records/<project>/<run>/` | 样本/参数、命令、日志、状态、资源、清单和验收 |
| `01_software/`、`envs/` | 读者实现的代码、配置、实际环境锁和环境 |

项目、运行和分析文库三个编号分别管理。改变样本关系、参数、代码、环境或参考时创建新运行，保留前版本。运行记录至少保存输入 SHA-256、参数全文、实现代码哈希、环境/外部依赖身份、参考哈希和产物清单。

## 4. 安装环境并实际验证

### 4.1 Miniforge 与隔离设置

从 [Miniforge 官方仓库](https://github.com/conda-forge/miniforge) 选择系统/架构匹配的安装器。下面固定一个实践基线版本；若官方版本不可得或目标系统不兼容，记录替代来源、版本和原因，不能悄悄改为 latest。

```bash
set -euo pipefail
export CT_MINIFORGE_VERSION='26.7.2-0'
CT_OS="$(uname -s)"
CT_ARCH="$(uname -m)"
case "$CT_OS/$CT_ARCH" in
  Darwin/arm64|Darwin/x86_64|Linux/x86_64|Linux/aarch64) ;;
  *) printf '请核对当前系统与官方安装器架构\n' >&2; exit 1 ;;
esac
if [ "$CT_OS" = Darwin ]; then CT_OS=MacOSX; fi
CT_INSTALLER="Miniforge3-${CT_MINIFORGE_VERSION}-${CT_OS}-${CT_ARCH}.sh"
CT_RELEASE="https://github.com/conda-forge/miniforge/releases/download/${CT_MINIFORGE_VERSION}"
curl -fL --retry 3 "$CT_RELEASE/$CT_INSTALLER" -o "$CT_ROOT/07_cache/downloads/$CT_INSTALLER"
curl -fL --retry 3 "$CT_RELEASE/$CT_INSTALLER.sha256" -o "$CT_ROOT/07_cache/downloads/$CT_INSTALLER.sha256"
(
  cd "$CT_ROOT/07_cache/downloads"
  shasum -a 256 -c "$CT_INSTALLER.sha256"
)
# 确认该前缀不存在；已有部署不执行这一条安装命令。
bash "$CT_ROOT/07_cache/downloads/$CT_INSTALLER" -b -p "$CT_ROOT/conda"
export CT_CONDA="$CT_ROOT/conda/bin/conda"
"$CT_CONDA" --version
```

Linux 若没有 `shasum`，用系统 `sha256sum -c` 校验同一文件；不要跳过校验。保存安装器及校验来源。本机历史 ARM 安装器 SHA-256 为 `d70bfa2e97afcda96927c9b9ca0e2316cb7750e4ce651c94388267cbe9588711`，它仅对应这个固定版本的 Mac ARM 文件。

项目内配置与调用示例：

```bash
cat > "$CT_ROOT/01_software/config/condarc.yaml" <<'YAML'
channels:
  - conda-forge
  - bioconda
channel_priority: strict
auto_activate_base: false
YAML
export CONDARC="$CT_ROOT/01_software/config/condarc.yaml"
ctrun() {
  local ct_module="$1"
  shift
  env -u PYTHONPATH -u R_HOME -u R_LIBS_USER -u JAVA_HOME -u JAVA_LD_LIBRARY_PATH -u LC_ALL \
    "$CT_CONDA" run --no-capture-output -p "$CT_ROOT/envs/$ct_module" "$@"
}
```

函数示例通过 Conda 激活相应进程环境，避免多个 Python/R 混用。将函数与变量保存到读者自己的入口文件；子进程需要的设置也要明确传递。项目设置不等于第三方程序在整个系统零目录外写入。

### 4.2 按模块安装

以下是**需求配方示例**，不是已求解的跨平台锁，也不要求所有模块一次安装。先做核心流程，确有需要再安装 motif 或其他扩展。求解冲突时记录缺失包/构建，选择受支持的组合并验证。

```bash
"$CT_CONDA" create -y -p "$CT_ROOT/envs/control" --override-channels -c conda-forge -c bioconda \
  python=3.12 pandas pyyaml psutil matplotlib jinja2 openpyxl
"$CT_CONDA" create -y -p "$CT_ROOT/envs/qc" --override-channels -c conda-forge -c bioconda \
  python=3.12 fastqc cutadapt seqkit openjdk=17
"$CT_CONDA" create -y -p "$CT_ROOT/envs/alignment" --override-channels -c conda-forge -c bioconda \
  bowtie2 samtools bedtools picard openjdk=17
"$CT_CONDA" create -y -p "$CT_ROOT/envs/peaks" --override-channels -c conda-forge -c bioconda \
  python=3.12 macs3
"$CT_CONDA" create -y -p "$CT_ROOT/envs/signal" --override-channels -c conda-forge -c bioconda \
  python=3.12 deeptools multiqc pysam pybigwig numpy scipy
"$CT_CONDA" create -y -p "$CT_ROOT/envs/r_analysis" --override-channels -c conda-forge -c bioconda \
  r-base=4.5 bioconductor-chipseeker=1.46 bioconductor-clusterprofiler=4.18 \
  bioconductor-txdbmaker=1.6 bioconductor-genomicfeatures bioconductor-genomicranges \
  bioconductor-rtracklayer bioconductor-annotationdbi bioconductor-org.hs.eg.db=3.22 \
  bioconductor-genomeinfodbdata r-jsonlite r-ggplot2
# 可选：同时核对 HOMER 的 known motif 数据来源，不只检查脚本存在。
"$CT_CONDA" create -y -p "$CT_ROOT/envs/motif" --override-channels -c conda-forge -c bioconda homer
```

R 配方以本机 R 4.5 / Bioconductor 3.22 兼容系列为参考；新机器不混用不同系列，其他物种增加其匹配的 OrgDb。若改走 BiocManager，使用独立前缀/包库和该 R 对应的官方 Bioconductor 版本，保存完整会话；不要为排错在同一环境反复混装。参见 [Bioconductor](https://bioconductor.org/install/)。

本机曾使用的精选版本：Python 3.12.15、Cutadapt 5.2、Bowtie2 2.5.5、Samtools 1.24、bedtools 2.31.1、Picard 3.5.0、MACS3 3.0.5、deepTools 4.0.0、MultiQC 1.35、HOMER 5.1、R 4.5.3、ChIPseeker 1.46.1、clusterProfiler 4.18.4。它们是历史部署背景，非在线最新推荐或本目录附带的二进制版本。

### 4.3 部署验收与锁定

```bash
ctrun qc fastqc --version
ctrun qc cutadapt --version
ctrun alignment bowtie2 --version
ctrun alignment samtools --version
ctrun peaks macs3 --version
ctrun signal python -c 'import pysam,pyBigWig,numpy,scipy; print("signal imports OK")'
ctrun control python -c 'import pandas,yaml,openpyxl,jinja2; print("report imports OK")'
ctrun r_analysis Rscript -e 'for(p in c("ChIPseeker","txdbmaker","clusterProfiler","org.Hs.eg.db","GenomeInfoDbData")) library(p,character.only=TRUE); sessionInfo()'
```

版本/包加载之后，还要实际执行小型 FastQC、修剪、索引、比对、重复标记、成对处理、峰、轨道和报告；R 要真实生成并重载小型 TxDb。可选模块执行其小测试。仅 `--version` 成功不能证明完整工具链运行。

Java 找不到时检查模块是否有 `lib/jvm/bin/java`，用模块内实际 Java 设置该进程的 `JAVA_HOME` 与 PATH。ARM 的原生压缩库 warning 先结合退出码、回退行为和产物判断；必要时使用安装版本支持的 JDK 压缩选项，并另记影响。不要全量重装来处理一个 warning。

对实际已安装并验证的每个模块导出 explicit 和 from-history；导出目录及文件名应属于本次部署记录，不覆盖旧锁：

```bash
CT_LOCK_DIR="$CT_ROOT/06_records/deployment_run01/locks"
mkdir -p "$CT_LOCK_DIR"
for ct_module in control qc alignment peaks signal r_analysis; do
  "$CT_CONDA" list --explicit -p "$CT_ROOT/envs/$ct_module" > "$CT_LOCK_DIR/$ct_module.explicit.txt"
  "$CT_CONDA" env export --from-history -p "$CT_ROOT/envs/$ct_module" > "$CT_LOCK_DIR/$ct_module.history.yaml"
done
```

同平台重建可使用实际 explicit；跨平台按需求重新求解并验证。Conda 前缀有绝对路径，缓存和环境可能共享硬链接/软链接；不要直接拖动环境或删除整份解包缓存。[官方重建说明](https://docs.conda.io/projects/conda/en/stable/user-guide/tasks/manage-environments.html#explicit-spec-files)。

## 5. 准备并冻结参考资源

### 5.1 参考合同

本机真实人源基线是 **GRCh38 primary assembly / GENCODE v49**。primary assembly 含其所包含的非标准支架，不能误写为只含主染色体，也不能与含全部 alternate/haplotype 的 FASTA 混用。[GENCODE v49](https://www.gencodegenes.org/human/release_49.html)。

每个参考版本至少登记：物种、assembly、annotation、下载 URL/日期、原文件校验、解压校验、序列名和长度、索引构建命令/版本、TxDb 来源、OrgDb、blacklist 和有效基因组大小。FASTA 改一个碱基也属于变化；更新另建参考版本并判断下游重算。

| 资源 | 完成条件 |
|---|---|
| FASTA 与 `.fai` | 下载/解压校验通过，序列名唯一、长度合法 |
| GTF | 同一 assembly，染色体/区间在所选参考范围内 |
| Bowtie2 索引 | 全套文件存在，实际小型比对通过；记录源 FASTA 哈希 |
| `chrom.sizes` | 从实际 FASTA 的 `.fai` 导出 |
| blacklist | 来源、版本、适用 assembly 明确，名称/区间核对通过 |
| TxDb / OrgDb | TxDb 来自同版 GTF；OrgDb 匹配物种并记录版本 |
| TSS/基因体 BED6 | 从该 GTF 按明确 gene/transcript 口径生成，保留方向 |
| spike-in（如用） | 读者自己的已确认序列、唯一名称、来源和哈希；无实验依据时不做校准 |

### 5.2 人源下载和构建示例

```bash
export CT_REF="$CT_ROOT/03_references/human/GRCh38_GENCODE49"
mkdir -p "$CT_REF/bowtie2"
CT_GENCODE='https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_49'
curl -fL --retry 3 "$CT_GENCODE/GRCh38.primary_assembly.genome.fa.gz" -o "$CT_REF/host.fa.gz"
curl -fL --retry 3 "$CT_GENCODE/gencode.v49.primary_assembly.annotation.gtf.gz" -o "$CT_REF/genes.gtf.gz"
gzip -t "$CT_REF/host.fa.gz" "$CT_REF/genes.gtf.gz"
shasum -a 256 "$CT_REF/host.fa.gz" "$CT_REF/genes.gtf.gz"
# 核对下面的固定版本 SHA-256 后，才继续解压与构建。
gzip -dc "$CT_REF/host.fa.gz" > "$CT_REF/host.fa"
ctrun alignment samtools faidx "$CT_REF/host.fa"
cut -f1,2 "$CT_REF/host.fa.fai" > "$CT_REF/chrom.sizes"
ctrun alignment bowtie2-build --threads 4 "$CT_REF/host.fa" "$CT_REF/bowtie2/host"
```

本机保存的上述原始 GENCODE 文件 SHA-256：

```text
host.fa.gz   3023c0705e83e86309271b09e25bdf678945fac98ce2fe5bb6b6e068f2225452
genes.gtf.gz 5cabe88e93064851dc6cb826efe0e8b7babf7985e73df280cfb05910d05cd023
```

这两项只适用于指定 release 和文件。若上游内容/校验不符，先定位原因，不用新算的值替换来源值使检查通过。下载恢复先写 `.partial`，校验后再作为正式资源；分段恢复必须核对区段与最终全文件校验，不能拼接任意失败片段。

blacklist 可使用 [Boyle-Lab 固定提交的 hg38 v2 文件](https://raw.githubusercontent.com/Boyle-Lab/Blacklist/61a04d2c5e49341d76735d485c61f0d1177d08a8/lists/hg38-blacklist.v2.bed.gz)。本机该压缩文件 SHA-256 是 `c92e763af17271446194991e71917ac220593a5a3d40a06667be24178ef08cf2`；解压后校验所有区间落在参考范围，不能把 hg38 blacklist 套给别的物种/assembly。

R 中使用 `txdbmaker::makeTxDbFromGFF()` 从同版 GTF 创建 TxDb，保存为 SQLite，再用 `AnnotationDbi::loadDb()` 实际重载。区域文件由 AI 实现时至少满足：

- 明确取 gene feature 还是 transcript feature；同一文件不混入多种统计单位。
- GTF 的 1-based inclusive 转 BED：`start_bed = start_gtf - 1`，`end_bed = end_gtf`。
- 保存 `chrom/start/end/gene_id/score/strand`，剔除或报告非法区间，不靠截断掩盖参考不匹配。
- 全基因与 protein_coding 区域分开；采用哪些染色体和基因类型写进方法。
- gene-level BED 的端点与 transcript-level TxDb TSS 不是同一集合，报告分别说明。

### 5.3 spike-in 与换物种

未使用外源时直接使用宿主参考；检测外源时，把读者已确认的外源序列加到宿主 FASTA，确保名称不冲突，另建联合索引、长度表和来源清单。外源序列改变须重建索引。本文不附作者的自定义序列，也不把未知来源短序列当作物种参考。

更换物种时重新准备 FASTA/GTF、区域、索引、blacklist、TxDb/OrgDb、有效基因组大小、motif 背景与通路映射。登记“待准备”与“验证就绪”两种状态；名称登记完成不能代替资源构建。

## 6. 登记样本并核验输入

### 6.1 身份优先于文件名

使用 [样本表](templates/samples.tsv)，一行一份 R1/R2 数据；[示例](templates/samples.example.tsv) 中的名称和关系均为虚构。

| 实验关系 | 登记/处理 |
|---|---|
| 同一原始文库加测 | 同一 library_id；不同 lane/run；联合重复标记 |
| 同一生物样品重新建库 | 不同 library_id，可共用 biological_sample_id；不自动新增生物学重复 |
| 独立生物样品 | 各自 biological_sample_id 和 library_id；重复关系由实验记录确认 |
| IgG | role=igg，自己的 control_id 留空；treatment 引用存在的 IgG library_id |

靶标、角色、组别、重复和校准依据未知时写明待确认。文件同名前缀不证明同库；不同处理组的 replicate=1 不代表同一分析单位；IgG 不能当作 treatment 的独立重复。

同库多行的角色、靶标、生物样品、组别、实验批次、对照和 spike-in 依据应相容；同一路径重复登记、lane/run 冲突或无法确认同库时，停止合并。合并前保存“来源行 → 分析文库”的预览和确认依据。

### 6.2 输入预检实现合同

对每份压缩原件保存来源、文件名、大小、接收时间、来源已有哈希和本机 SHA-256。先核对已有清单；仅对复制后的文件新算哈希不能证明与原件相同。不得覆盖、改写或为省空间先删除原件。

AI 应编写流式预检程序，逐对读取完整 R1/R2（不是只读开头），检查：

1. gzip 完整、FASTQ 记录完整、`@`/`+` 合法，序列与质量长度一致。
2. R1/R2 同时到达 EOF，记录数相同；每条为预期双端数据。
3. 依据实际测序格式规范化 mate 标记后 read ID 相同；不任意截断读名制造匹配。
4. 输出 pairs、两端 reads、精确 bases、长度分布和 SHA-256；错误包含文件、记录位置和原因。
5. 异常/重复 read ID 记录统计；同名主记录导致后续成对分组不唯一时必须解决，不能静默丢弃。

预检量纲：`reads = 2 × pairs` 仅对完整的双端集合成立；bases 使用实际序列长度累加，不能用固定 PE150 推算修剪后碱基量。Mb/Gb 明确采用十进制碱基单位。

### 6.3 同库加测实现合同

各来源单独保留预检、原始 FastQC 和校验。合并前给两端相同的 read ID 核心加上来源命名空间；保存原始 ID 对应与来源清单，防止不同测序批次同名串配。

可每 lane 单独修剪/比对后合并 BAM，也可命名隔离后统一修剪/比对；两种路线都必须保留来源、每 lane QC 和 RG/PU，且同一真实文库共用 LB。所有 lane 汇总后联合 MarkDuplicates，再进行最终过滤。不要先各 lane 去重再拼接，也不要把生物学重复合成一个样本。

## 7. QC、修剪、比对与重复标记

### 7.1 示例运行上下文

以下命令针对一份虚构文库的一个 lane。先替换真实输入、确认参考索引选择，并为每次运行使用新的目录；批量执行由读者编写循环/调度，不把示例当成自动发现工具。

```bash
export CT_PROJECT='demo_project'
export CT_RUN='run01'
export CT_LIBRARY='TREAT_A'
export CT_LANE='RUN_A_L001'
export CT_THREADS=4
export CT_WORK="$CT_ROOT/04_intermediate/$CT_PROJECT/$CT_RUN/$CT_LIBRARY"
export CT_LOG="$CT_ROOT/06_records/$CT_PROJECT/$CT_RUN/$CT_LIBRARY"
export CT_R1="$CT_ROOT/02_input/$CT_PROJECT/TREAT_A_L001_R1.fastq.gz"
export CT_R2="$CT_ROOT/02_input/$CT_PROJECT/TREAT_A_L001_R2.fastq.gz"
# 无外源时使用宿主索引；使用外源时换为已核验的联合索引。
export CT_INDEX="$CT_REF/bowtie2/host"
mkdir -p "$CT_WORK"/{raw_qc,clean_qc,alignment,filter,peaks,signal,annotation} "$CT_LOG"
```

### 7.2 FastQC 与 Cutadapt

```bash
ctrun qc fastqc -t 2 -o "$CT_WORK/raw_qc" "$CT_R1" "$CT_R2" > "$CT_LOG/fastqc_raw.log" 2>&1
# 这只是曾用过的接头示例；执行前按试剂盒/建库资料确认两端序列。
export CT_ADAPTER_R1='CTGTCTCTTATACACATCT'
export CT_ADAPTER_R2='CTGTCTCTTATACACATCT'
ctrun qc cutadapt -j "$CT_THREADS" -a "$CT_ADAPTER_R1" -A "$CT_ADAPTER_R2" \
  -q 20 -m 30 --max-n 0.1 --pair-filter any \
  --json "$CT_WORK/cutadapt.json" \
  -o "$CT_WORK/clean_R1.fastq.gz" -p "$CT_WORK/clean_R2.fastq.gz" \
  "$CT_R1" "$CT_R2" > "$CT_LOG/cutadapt.log" 2>&1
ctrun qc fastqc -t 2 -o "$CT_WORK/clean_qc" \
  "$CT_WORK/clean_R1.fastq.gz" "$CT_WORK/clean_R2.fastq.gz" > "$CT_LOG/fastqc_clean.log" 2>&1
```

检查 Cutadapt JSON 的输入/通过/丢弃 pair 记账，并重新核对 clean 配对完整性。关闭修剪时保留原因与实际输入位置，不能要求不存在的 Cutadapt 统计或伪造 clean 改善。参数语义以 [Cutadapt 官方文档](https://cutadapt.readthedocs.io/en/stable/guide.html) 和本机版本帮助为准。

FastQC 的 WARN/FAIL 是模块状态，程序退出失败是另一类问题；CUT&Tag 的序列组成和重复度应结合靶标、片段、比对与对照解释，不以单个模块判定整个实验成功或失败。

### 7.3 比对并保留未比对记录

实践起点：very-sensitive、end-to-end、允许 dovetail、禁止 discordant、插入范围 10–1000 bp、固定种子。端到端与局部比对、短片段和上限按项目确定。下面保留 Bowtie2 默认 mixed 搜索，以便单端支持另列；不要无记录地在不同批次切换。[Bowtie2 手册](https://bowtie-bio.sourceforge.net/bowtie2/manual.shtml)。

```bash
set -euo pipefail
ctrun alignment bowtie2 --very-sensitive --end-to-end --dovetail --no-discordant \
  -I 10 -X 1000 -p "$CT_THREADS" --seed 20261015 \
  --rg-id "$CT_LIBRARY.$CT_LANE" --rg "SM:$CT_LIBRARY" --rg "LB:$CT_LIBRARY" \
  --rg "PU:$CT_LANE" --rg 'PL:ILLUMINA' \
  -x "$CT_INDEX" -1 "$CT_WORK/clean_R1.fastq.gz" -2 "$CT_WORK/clean_R2.fastq.gz" \
  2> "$CT_LOG/bowtie2.log" | \
  ctrun alignment samtools sort -@ 2 -m 512M -T "$CT_WORK/alignment/sorttmp" \
    -o "$CT_WORK/alignment/competitive.bam" - 2> "$CT_LOG/samtools_sort.log"
ctrun alignment samtools quickcheck -v "$CT_WORK/alignment/competitive.bam"
ctrun alignment samtools flagstat "$CT_WORK/alignment/competitive.bam" > "$CT_LOG/competitive_flagstat.txt"
```

管道使用 pipefail，执行程序应分别记录 Bowtie2 与排序退出码/日志，不能只看最后一个进程。samtools 的 `-m` 是每线程内存预算，不是整个管道内存。[Samtools 文档](https://www.htslib.org/doc/samtools.html)。

多 lane 时，命名空间须在比对前完成，RG/PU 各不相同、LB 相同，先合并并坐标排序，再执行下节。不要对本例中的单 lane BAM 单独标记后再当作同库最终结果。

### 7.4 标记重复

```bash
ctrun alignment picard -Xmx4g MarkDuplicates \
  I="$CT_WORK/alignment/competitive.bam" O="$CT_WORK/alignment/marked.bam" \
  M="$CT_WORK/alignment/duplicate_metrics.txt" REMOVE_DUPLICATES=false \
  TMP_DIR="$CT_WORK/alignment" > "$CT_LOG/markduplicates.log" 2>&1
ctrun alignment samtools sort -n -@ 2 -m 512M -T "$CT_WORK/alignment/namesort" \
  -o "$CT_WORK/alignment/marked.name.bam" "$CT_WORK/alignment/marked.bam"
```

先标记并保留，再按记录的政策决定最终过滤是否删除。坐标重复不自动证明 PCR 原因；CUT&Tag 的保留/去除应结合靶标和设计，必要时另建敏感性分析版本。

如果给读名增加了来源前缀，先验证 Picard 光学重复解析规则是否仍能正确取 tile/x/y；不支持时明确“光学重复不可评估”，普通重复标记与光学统计分开。不要从读名解析 warning 推断普通重复全部错误。选项详见 [Picard](https://broadinstitute.github.io/picard/command-line-overview.html)。

## 8. 实现严格的成对过滤与片段记账

### 8.1 输入、输出与算法合同

这是需要 AI 实现并测试的关键模块。输入为名称排序后的重复标记 BAM、参考/外源/线粒体名、blacklist、MAPQ/长度/重复政策；使用 pysam 等读取。不能把逐 read 的 `samtools view -q` 结果直接当作完整片段集合。

按带来源命名空间的 query name 分组，仅保留 primary records 作配对判断；secondary/supplementary 另外统计。每组必须恰好一条 R1、一条 R2 主记录。缺 mate、多主记录或来源串配属于输入/实现错误，应停止并定位，不能静默删除以凑齐统计。

对每个完整 pair，按固定优先级分配**唯一终止类别**：

| 顺序 | 类别 | 判定 |
|---|---|---|
| 1 | unmapped | 两端均未比对 |
| 2 | single_mapped | 仅一端比对 |
| 3 | non_concordant | 参考不一致或任一端不是 proper pair |
| 4 | qcfail | 任一端带 QC fail |
| 5 | low_mapq | 两端 MAPQ 的最小值低于阈值 |
| 6 | fragment_length | 完整片段外边界长度不在范围内 |
| 7 | duplicate_removed | 本运行选择去重复，且任一端被标记重复 |
| 8 | spikein | 满足前述要求并落在登记的外源序列 |
| 9 | mitochondria | 启用排除且属于登记的线粒体序列 |
| 10 | blacklist | 完整片段与 blacklist 有至少 1 bp 重叠 |
| 11 | retained_host | 通过以上筛选的宿主片段 |

完整片段采用 0-based 半开区间：`start = min(reference_start_R1, reference_start_R2)`，`end = max(reference_end_R1, reference_end_R2)`。核对 `end-start` 与 proper pair 的 TLEN、mate 位置和方向一致；异常先定位比对/CIGAR/读名，不悄悄使用单端长度替代。

blacklist 先按染色体合并区间，判断 `fragment_start < blacklist_end` 且 `fragment_end > blacklist_start`。相邻但不相交不算命中。低 MAPQ、重复标记、外源等独立诊断计数可额外保存，但不能与终止类别混加。

输出要求：

- 宿主 BAM 同时写出一对的两个 primary records，保留 RG/重复标记；随后坐标排序、建 BAI 并 quickcheck。
- `fragments.bed` 一行一个合格宿主片段；含 chrom/start/end/唯一 ID，允许重复坐标，不能按坐标无条件去重。
- 保存 clean_pairs、全部互斥类别、合格外源及其重复子集、宿主长度分布、过滤政策和参考身份。
- `sum(11 个互斥类别) == clean_pairs`。
- `宿主 primary records == 2 × retained_host`，`fragments.bed 行数 == retained_host`。
- 数据结构或分类不闭合即失败；不能靠修改统计 JSON 通过检查。

重复保留时，标记重复仍进入其适用的宿主/外源类别，另计子集。重复去除时按上述顺序先进入 duplicate_removed；宿主与外源计数政策保持一致。任意改变分类顺序、阈值或外源政策，都要另建分析版本。

### 8.2 必做的小型边界测试

构造 11 对已知主记录：2 对可保留宿主（其中 1 对标记重复），其余分别属于 unmapped、single_mapped、non_concordant、qcfail、low_mapq、fragment_length、spikein、mitochondria、blacklist 各 1 对。

| 测试 | 预期 |
|---|---|
| retain duplicates | clean=11，宿主=2，外源=1，其余 8 个终止类别各 1，duplicate_removed=0 |
| remove duplicates | 宿主=1，duplicate_removed=1，其余相同，类别仍合计 11 |
| 孤立 mate / 同名多主记录 | 明确失败，包含相关 ID |
| blacklist 首尾相接 / 交叠 1 bp | 前者不命中，后者命中 |
| 两 lane 读名相同 | 命名空间后分组独立，RG/PU 不同而 LB 相同 |

MAPQ 和长度阈值要增加“恰好等于边界”的正例；dovetail/overlap、方向、spike 重复和空输入另做测试。合成测试证明实现按合同工作，不能证明真实样本质量或校准有效。

### 8.3 无有效宿主片段

保留输入、QC、比对和过滤记账，结果标明无有效宿主。CPM、FRiP、峰、TSS 和基因关联不能继续生成假图或假数值。BAM/BAI 的结构核对与“非零有效片段”是不同条件；记录哪些产物存在、哪些模块不适用。

## 9. spike-in：加入、计数与校准分别判断

先确认实验是否加入、何时加入、序列与投入量、不同样本是否可比、重复政策和最低有效量，再决定是否校准。出现外源命中只是检测证据；短序列、高重复或低唯一性计数不能自动成为定量标准。

基线路径默认关闭校准，按完整 proper pair、两端 MAPQ/长度和同一重复政策计数。若实验允许校准，保存合格外源数、重复子集、支持范围和依据；不足时记录“仅计数/不具资格”。阈值如 100 个片段只是本机实现起点，没有跨实验的普适合格意义。

显示轨道缩放可以定义 `scale = 目标外源片段数 / 本样本合格外源片段数`，并从未归一化的宿主片段轨道应用一次。保存目标值、原始计数、公式和单位；不要在 CPM 上重复套公式而不说明。保留未校准 CPM 便于核对。

此类轨道缩放不自动定义差异分析的归一化。真实绝对定量和组间比较需要对应实验设计与方法验证，本文不提供已验证的差异统计流程。

## 10. 峰、FRiP 与片段信号

### 10.1 MACS3 BAMPE

处理样本和配对 IgG 用同一参考与过滤政策；如果使用 IgG，将其合格宿主 BAM 作为 control。IgG 本身不要求生成 treatment 峰。没有 IgG 时明确“无外部背景对照”，不能悄悄把别的处理样本当对照。

```bash
export CT_HOST_BAM="$CT_WORK/filter/analysis.bam"
export CT_GSIZE=2913022398
# 以下示例未传 -c，适用于明确选择无对照的运行。
# 有匹配 IgG 时，在两个 callpeak 中均增加 -c "$CT_CONTROL_BAM"，并保存其身份。
ctrun peaks macs3 callpeak -t "$CT_HOST_BAM" -f BAMPE --keep-dup all \
  -g "$CT_GSIZE" -q 0.01 -n narrow --outdir "$CT_WORK/peaks/narrow" \
  > "$CT_LOG/macs3_narrow.log" 2>&1
ctrun peaks macs3 callpeak -t "$CT_HOST_BAM" -f BAMPE --keep-dup all \
  -g "$CT_GSIZE" -q 0.01 --broad --broad-cutoff 0.1 \
  -n broad --outdir "$CT_WORK/peaks/broad" > "$CT_LOG/macs3_broad.log" 2>&1
```

这里的 effective genome size 对应所选人源基线，换物种/assembly 重新选择并记录。重复已由上游政策控制，`--keep-dup all` 避免 MACS3 再实施另一套坐标截断。BAMPE 使用真实片段，不能无依据复制单端 ATAC/ChIP 示例的 shift/extsize。[MACS3 官方说明](https://macs3-project.github.io/MACS/docs/callpeak.html)。

narrow/broad 按靶标与方法选择；同时生成时分别解释，不能按峰多、FRiP 大就选择“质量更好”的模式。零峰可为合法结果，保留工具日志和方法，不捏造峰供下游展示。

### 10.2 FRiP 的唯一分母

本指南的自身峰 FRiP 定义：

`FRiP = 与该模式任一峰重叠至少 1 bp 的合格宿主片段数 / 同分支全部合格宿主片段数`。

一个片段即使重叠多个峰也只计一次。分母为 fragment，不能混成 BAM reads、raw pairs 或所有 mapped reads；不同模式的峰范围不同。跨样本若使用共同峰集合，需要另行记录该集合和方法，不能与自身峰 FRiP 混比。

独立复核示例使用 [bedtools intersect](https://bedtools.readthedocs.io/en/stable/content/tools/intersect.html)：

```bash
ctrun alignment bedtools intersect -u -a "$CT_WORK/filter/fragments.bed" \
  -b "$CT_WORK/peaks/narrow/narrow_peaks.narrowPeak" > "$CT_WORK/peaks/narrow/fragments_in_peaks.bed"
wc -l "$CT_WORK/filter/fragments.bed" "$CT_WORK/peaks/narrow/fragments_in_peaks.bed"
```

分母>0 时检查 `0 ≤ 分子 ≤ 分母`；零峰时分子为 0，分母为 0 时记 NA/不可评估。构造相交峰用例验证分子不会重复累加。

### 10.3 轨道统计合同

作者本机采用**每个完整片段对其重叠的每个 bin 计一次**，例如 25 bp bin。AI 实现时以 `fragments.bed` 为来源，按参考长度建立 bin，`floor(start/bin)` 到 `floor((end-1)/bin)` 都加 1；末 bin 截断到染色体长度。输出原始 bin 计数或可重建摘要，并用 pyBigWig 写轨道。

`fragment_CPM(bin) = 该 bin 的片段重叠数 × 1,000,000 / 全部合格宿主片段数`。

外源与按政策排除的线粒体不进宿主轨道；保留的支架不能悄悄从轨道或分母消失。零覆盖与无数据范围分别处理，头部与参考顺序/长度一致。记录 bin、边界、归一化、过滤和参考身份。

已知答案测试：宿主只有 `[10,40)` 与 `[30,60)` 两个片段，bin=25，则前三个 bin 的重叠数为 `1,2,1`，CPM 为 `500000,1000000,500000`；再从生成的 bigWig 实读核对，允许浮点存储误差。增加 dovetail/overlap、末 bin 和全零染色体测试。

另一个可选路线是用 [bedtools genomecov](https://bedtools.readthedocs.io/en/latest/content/tools/genomecov.html) 对片段 BED 生成**逐碱基覆盖**：

```bash
ctrun alignment bedtools sort -i "$CT_WORK/filter/fragments.bed" -g "$CT_REF/chrom.sizes" \
  > "$CT_WORK/filter/fragments.sorted.bed"
ctrun alignment bedtools genomecov -bg -i "$CT_WORK/filter/fragments.sorted.bed" \
  -g "$CT_REF/chrom.sizes" > "$CT_WORK/signal/fragment_base_coverage.bedgraph"
```

该 BedGraph 还需要按同一片段分母缩放并转换为 bigWig。逐碱基覆盖、bin 平均碱基覆盖和“每个重叠 bin 计一次”是不同统计口径；选择一种并写入方法，不用它们逐值互相验收。不能默认一个 bamCoverage 参数组合与上述 bin 合同一致，尤其要用短片段和 dovetail 用例核对。

### 10.4 片段长度、复杂度与 Fingerprint

从完整片段生成长度直方图；重复坐标统计保存总片段 N、唯一坐标数 U、出现一次的坐标数 M1 和两次的 M2。可报告 `NRF=U/N`、`PBC1=M1/U`、`PBC2=M1/M2`；分母为零时 NA。它们反映此处坐标口径，不直接等于 PCR 重复率或预期补测收益。

Fingerprint 可按固定基因组窗口统计片段重叠数，再绘制按信号排序的累计窗口比例与累计信号比例；保存窗口集合、零/N 区域政策、bin、直方图和分母。不同窗口范围/过滤定义不直接横比，不把一条曲线当靶标特异性的充分证据。没有 Preseq 结果就不画推测的 Preseq 曲线。

## 11. TSS、基因体与多样本探索

### 11.1 矩阵和图

实际区域模式选择 protein_coding、all_genes 或自定义 BED，启用前检查对应文件与参考版本。下面示例采用 gene-level 带方向 BED6；没有提供该文件时，由第 5 节的转换合同生成并验证。

```bash
export CT_TRACK="$CT_WORK/signal/fragment_CPM.bw"
export CT_REGIONS="$CT_REF/protein_coding_genes.bed"
ctrun signal computeMatrix reference-point --referencePoint TSS \
  -b 3000 -a 3000 --binSize 25 -R "$CT_REGIONS" -S "$CT_TRACK" \
  --missingDataAsZero --sortRegions keep --numberOfProcessors "$CT_THREADS" \
  -o "$CT_WORK/signal/tss_matrix.gz" > "$CT_LOG/tss_matrix.log" 2>&1
ctrun signal computeMatrix scale-regions -b 3000 -a 3000 --regionBodyLength 5000 \
  --binSize 50 -R "$CT_REGIONS" -S "$CT_TRACK" --missingDataAsZero --sortRegions keep \
  --numberOfProcessors "$CT_THREADS" -o "$CT_WORK/signal/gene_body_matrix.gz" \
  > "$CT_LOG/gene_body_matrix.log" 2>&1
ctrun signal plotProfile -m "$CT_WORK/signal/tss_matrix.gz" -out "$CT_WORK/signal/tss_profile.png"
ctrun signal plotHeatmap -m "$CT_WORK/signal/tss_matrix.gz" -out "$CT_WORK/signal/tss_heatmap.png"
```

这些参数是示例；更改区域、bin、上下游范围或 missing-data 政策需记录。`missingDataAsZero` 仅在缺失确可按该方法处理的匹配轨道上使用，不能修复组装不匹配。短基因/区域跳过 warning 保存原义和实际范围；不要改基因集合只为图好看。[computeMatrix](https://deeptools.readthedocs.io/en/latest/content/tools/computeMatrix.html)。

图必须注明信号单位、TSS 定义、区域数、排序和色标。同一比较使用共同色标/轴范围；展示缩图不改变矩阵、信号 bin 或科学排序。保存原矩阵、原图和绘图参数。

### 11.2 相关性与 PCA 的实施方向

只比较身份已确认、同靶标且方法可比的独立样本；lane、加测、技术重建库与 IgG 不自动成为独立生物学重复。使用未做 spike-in 显示缩放的相同 Fragment CPM，分别按靶标处理。

一个可实现的起点：标准核染色体 10 kb 非重叠窗口，排除 blacklist；只使用所有轨道具有完整范围且均为有限值的共同窗口，保留 BED、矩阵和各类排除计数。缺失不自动填零；可记录并排除全样本均零窗口。

至少两个可比样本/有效窗口计算 Spearman，常量样本系数记 NA。PCA 可要求至少三个样本、足够可变窗口且秩支持两维；对信号 log1p 后按窗口跨样本中心化，保存是否方差缩放、坐标和解释方差。这个样本数要求是此实践的展示规则，不是 PCA 算法的普遍下限。

相关性/PCA 是探索性 QC，不设置跨靶标万能合格线，不自动证明重复独立或完成差异峰统计。读者可采用 [deepTools 相关性](https://deeptools.readthedocs.io/en/latest/content/tools/plotCorrelation.html) / [PCA](https://deeptools.readthedocs.io/en/latest/content/tools/plotPCA.html) 或自行实现，但要核对其矩阵和缺失处理是否符合所选合同。

## 12. 峰注释、GO、KEGG 与 known motif

### 12.1 峰注释的全行保留

使用来自同版 GTF 的 TxDb 和匹配物种 OrgDb，采用 [ChIPseeker](https://bioconductor.org/packages/3.22/bioc/html/ChIPseeker.html) 等工具。保存 promoter 范围、TSS/transcript 口径、序列范围、包版本和 R sessionInfo。

AI 实现必须以**稳定峰 ID**映射原峰与注释，不能仅凭坐标连接而丢失重复坐标的独立行。原始 BED 为 0-based 半开，注释表按实际工具坐标说明转换；保留原坐标与注释坐标基准。

没有相应基因特征的支架峰、无关联基因或无法映射的条目仍保留为 Unannotated/未映射，并记录原因。`原峰数 == 完整注释行数`；不能静默丢行改善饼图。功能区组成的数量/比例使用相同全峰集合，显示分母。

最近基因或 promoter 关联是候选关系，不证明该峰调控该基因，也不直接代表其表达变化。

### 12.2 GO：背景、映射与多重检验

先定义可参与的宿主核基因背景，保留 GTF 基因集合、Ensembl 去版本规则、OrgDb 映射、未映射与一对多清单。一个保守起点是只让唯一 Ensembl→Entrez 映射参与 ORA，目标/背景都用同一规则并去重，再取实际具有相应 GO 注释的集合。

使用 [clusterProfiler](https://bioconductor.org/packages/clusterProfiler/) 时明确 `universe`、keyType、ontology、基因集大小、P 值与多重检验政策。若三类 GO 联合 BH，必须设置/实现相应检验家族并独立核对，不能把各 ontology 已校正的 P 值再混称为联合结果。

输出目标/背景列表、映射统计、实际进入检验/校正的全部条目、显著表和方法说明。保存未格式化的 P 值来源；显著表应可由完整检验表按预设规则重建。

可抽样用超几何检验独立核对：`P(X >= k)`，记录总体背景数 N、通路/条目背景数 K、目标数 n、命中数 k，并重算所定义家族的 BH。没有显著项正常输出零条及原因，不只保留 Top 10。

### 12.3 KEGG：冻结快照与同一统计合同

先确认用途和数据来源；公共接口访问、下载快照与商业使用的条件按 [KEGG 官方说明](https://www.kegg.jp/kegg/legal.html) 核对。本文不分发 KEGG 数据，也不提供商业授权。

无论合法 API 快照或用户已有授权本地资料，都保存 organism、来源、取得日期、release、通路名称表、基因-通路映射、文件哈希及使用说明。创建分析时冻结该快照；重绘报告不自动更新数据库。

目标/背景去重并使用相同 Entrez 映射政策；记录可映射到此库的实际目标数/背景数。若采用全部合格大小通路作为检验家族，则包含零命中通路，并在该家族内 BH；不能只对显著/非零命中条目另行校正而省略方法差异。

完整表至少保存 ID、名称、GeneRatio、BgRatio、FoldEnrichment、pvalue、p.adjust、Count 和命中基因。ORA 说明峰关联基因的候选富集，不支持“通路激活/抑制”的直接结论。

### 12.4 known motif

按预定峰模式选区，例如对 narrow summit 截固定长度窗口；边界、标准染色体、blacklist 和缺失 summit 的处理都要记录。背景使用同参考、相容长度/GC/可比区域，保存来源和种子。

使用 [HOMER 官方资料](http://homer.ucsd.edu/homer/motif/) 及安装版本 `findMotifsGenome.pl` 帮助核对自定义 FASTA、`-size given`、`-nomotif` 和显式背景等选项。使用 known motif 集要保留数据库版本；脚本存在不等于数据已经准备。

输出完整 knownResults、原生 log P 值、目标/背景统计、序列/区域和方法。上游 q 显示为 0.0000 时保留原文及精度说明；GC 权重产生的非整数背景计数不强行取整。known motif 富集不证明某个转录因子已经结合；de novo 不属于本文已验证流程。

## 13. 生成报告并检查完整交付

### 13.1 先有可核对结果，再组织页面

MultiQC 可汇总工具 QC；本项目摘要、片段分母、完整注释和分析限制仍需读者/AI 根据保存结果另行组织。建议 HTML 模板用 Jinja2、绘图用已验证的绘图库；TSV 是完整数值底稿，Excel 可以用 openpyxl 导出并实际重读核对。公开路线不要求作者电脑中的专用 Node/artifact-tool 运行时；这些替代实现需读者自行验证。

```bash
CT_REPORT_DIR="$CT_ROOT/05_results/$CT_PROJECT/$CT_RUN"
mkdir -p "$CT_REPORT_DIR/01_QC"
ctrun signal multiqc "$CT_WORK" -o "$CT_REPORT_DIR/01_QC" \
  > "$CT_LOG/multiqc.log" 2>&1
```

主报告要显示本次项目/运行、物种/参考、分析文库、实际输入、角色/对照、过滤/重复政策、启用模块和限制。不要把本文或示例里的状态复制成真实结果。数据产生于哪次运行、是否复用/重绘/补算要能追溯。

### 13.2 建议的报告内容与目录

内部质控版重点放输入、QC、比对、成对过滤、长度/复杂度、轨道/TSS、峰、局部 IGV 和适用的重复性；交付版按实际开展情况增加完整注释、GO/KEGG/known motif 与概览。共有部分使用同一保存证据，切换版式不能改变科学结果。

```text
results/<project>/<run>/
  README.md
  report.html
  SHA256SUMS.txt
  report_assets/          预览、原图、完整表数据及本地样式/脚本
  01_QC/                 输入/修剪/比对与过滤摘要、FastQC、MultiQC
  02_BAM/<library>/      最终宿主 analysis.bam + analysis.bam.bai
  03_Signal/             bigWig、长度/复杂度、TSS/基因体图和方法
  04_Peaks/              所选模式完整峰及 FRiP
  05_Annotation/         全峰 TSV、可选 XLSX、组成与字段说明
  06_GO/                 全检验、显著项、目标/背景和方法（若开展）
  07_Motif/              完整 known motif、区域/背景和方法（若开展）
  08_Methods/            参数、版本、参考来源、单位和限制
  09_KEGG/               完整/显著检验及快照来源（若开展）
  10_Genome_Views/       IGV 选区/截图/会话/相对依赖（若开展）
  11_Diagnostics/        异常排查摘要（若开展）
  12_Reproducibility/    共同窗口、矩阵、图与方法（若开展）
```

这是建议组织方式，读者应建立自己的“必须/可选产物合同”。未启用目录可以省略或放明确说明；不能用空目录代表已分析。内部完整记录另存环境锁、代码、命令、状态、原始 JSON、绘图矩阵和来源清单；只读客户报告不需要继承所有排故材料。

### 13.3 大表、图片和 Excel

实践中主 HTML 一次内嵌大量原图与全表会显著膨胀。可以使用小主页面、缩图、原图和完整表附件，但复制时必须整体交付。

- Top 10 只是预览，保留全量记录；搜索/CSV 导出覆盖全部匹配行与原字段，“查看全部”可分页，例如每页 50 条。
- 坐标、基因名/ID 和极小 P 值按字段类型保存。统计原字符串不因 JSON 浮点或显示格式变成 0，基因名不被 Excel 自动转成日期。
- 默认排序公开：峰可按显著性降序，GO/KEGG 按校正 P 升序，known motif 按原生 log P 值；同分稳定排序，保留源表字段。
- 原图保留，缩图保持比例，提供放大/原图下载。缩图不平滑科学矩阵，不重新选择更“好看”的基因/峰。
- 对样本名、日志和表格字符串按文本输出/转义，不把输入内容直接当 HTML 或脚本执行。

若设计本地 `file://` 报告，注意普通 `fetch()` 读取附件可能受浏览器限制；可采用受控本地数据脚本或其他明确方案，并在目标浏览器实测。HTTP 模式通过不能写成 file 协议通过，DOM/文件哈希检查也不能代替真实下载验收。

### 13.4 IGV 与本地查看

从 [IGV 官方下载](https://igv.org/doc/desktop/DownloadPage/) 取得对应平台发行包及匹配 Java，不硬编码作者 macOS 应用目录。无桌面服务器应先做独立截图验证，选择合适的显示/虚拟显示方案；未解决时保留可用轨道/会话并明确截图未完成，不伪造截图。

会话保存 assembly、FASTA、BAM/BAI、bigWig、峰和基因区域的来源；客户包使用可移植的相对轨道依赖。远端 genome ID 首次打开可能联网，真正离线时需要匹配的本地参考及加载步骤，不能只写“hg38”就保证离线可用。

选区规则与 `selected_loci.tsv` 保留，例如显著峰和预先指定位点；多样本同坐标、同纵轴/下采样设置。区分 IGV read coverage 与完整片段 CPM。批处理命令参考 [IGV batch](https://igv.org/doc/desktop/UserGuide/tools/batch/)，选区截图仅是局部展示。

需要本地 HTTP 查看时可使用下面示例，执行终端需保持运行；它仅用于读者自己电脑的查看，不构成认证或公网部署方案：

```bash
ctrun control python -m http.server 8000 --bind 127.0.0.1 --directory "$CT_REPORT_DIR"
```

在浏览器打开对应本地地址，另行验收最终期望的打开方式。不要为查看报告把服务默认暴露到所有网卡。

### 13.5 完整包验收

同时核对 SHA-256、目录、相对链接、附件、BAM/BAI、轨道范围、峰与全表行数；Excel 实际打开或重读验证类型/行数，浏览器实际搜索 Top 10 之外的记录、翻到末页、导出 CSV、下载原图、打开 IGV 会话。

生成清单时列出整个冻结包（排除清单自身），在外部副本核对**复制前已有的清单**。报告通过不等于实验合格，文件 quickcheck 不等于所有片段统计通过。详细检查见 [验收清单](templates/ACCEPTANCE.md)。

## 14. 未比对 reads 的可选来源排查

默认不启用，不影响主流程。来源使用未过滤的宿主/外源联合比对 BAM；最终宿主 BAM 无法恢复已丢弃的未比对数据。保留未比对完整 pairs，R1/R2 单条 reads 作为抽样单位，全文件 reservoir 抽样，固定种子，合计最多 5000 条作为实践起点，不是 5000 对或两端各 5000 条。

AI 应实现导出 `queries.fasta`、匿名 query ID 清单、原 BAM/FASTA SHA-256、种子、实际抽样 reads、涉及 pairs 数和全部合格未比对 reads 数。重复序列保留抽样权重；没有合格序列时明确跳过。序列搜索的位置和范围按授权执行，不自动上传公共服务。

大核酸库优先在已准备数据库的服务器/存储上运行，先查当时官方体积和完整索引，不在本指南预设数据库已安装。Bowtie2 与 BLAST 索引不可互换。独立 BLAST+ 及数据库说明见 [NCBI BLAST Help](https://blast.ncbi.nlm.nih.gov/doc/blast-help/)。

一个外部搜索示例（先替换已准备的核酸库前缀）：

```bash
export CT_BLAST_DB='/absolute/path/to/nucleotide_database_prefix'
blastn -version > blast_version.txt
blastdbcmd -db "$CT_BLAST_DB" -info > database_info.txt
blastn -task blastn -query queries.fasta -db "$CT_BLAST_DB" \
  -num_threads 4 -dust yes -max_target_seqs 25 -max_hsps 1 \
  -outfmt '7 qseqid saccver staxids sscinames pident length qlen qstart qend evalue bitscore qcovhsp' \
  -out blast_hits.tsv
```

保存数据库快照/分类信息与版本，不把操作者填写的名字冒充程序独立验证。保留 outfmt 7 的 query 和处理总数声明；导入检查每条 query（包括零命中）、ID、长度、来源/种子和结果完整性，缺失/重复/陌生 ID 都失败。

判读可先用 identity≥80%、单 HSP query 覆盖≥70%、E-value≤1e-5，并保留最高 bitscore 5% 范围内竞争命中；这些仅是探索性起点。缺分类、多个 taxid 或达到命中上限时保留歧义。候选/歧义/无合格命中合计等于实际抽样数，比例分母也是实际抽样未比对 reads，不是命中行数或全库 reads。

抽样导出成功、真实搜索成功、完整导入和来源确认是四件事。受控命中表测试不代表真实检出；候选比例不等于已证实污染比例，不自动删除序列或更换主参考。确认需要更多数据、覆盖和实验来源。

## 15. 故障处理：先留证据，再做针对性修复

每次先保存项目/运行/步骤、时间、完整命令、退出码、原日志和当前产物。核对实际进程、可用空间、输入/参考和所用环境。日志暂时不更新不能单独证明卡死；历史失败不能因后来成功而被覆盖。

| 现象 | 优先检查与处置 | 恢复后验证 |
|---|---|---|
| 安装求解失败/包无对应架构 | 系统架构、channel、strict、包构建与 R/Bioc 兼容；保留求解日志，候选环境验证 | 实际版本、架构、工具链与新锁 |
| gzip 损坏、R1/R2 数量或 ID 不闭合 | 来源校验、传输、完整 EOF、lane 配对；恢复正确输入 | 全文件流式预检和来源已有哈希 |
| Java 找不到/系统占位入口 | 模块实际 Java、JAVA_HOME、激活环境；不随意换全局 Java | FastQC/Picard 小型实际运行 |
| ARM 原生压缩 warning | 实际是否回退、退出码、产物；必要时设置本版本支持的 JDK 兼容选项 | 正确产物与耗时记录，不能猜提速倍数 |
| R 包能加载但 GTF→TxDb 失败 | GenomeInfoDbData、GTF、R/Bioc 系列与日志；补匹配依赖 | 实际建库、保存、重载与坐标检查 |
| 某 Python 缺少 pandas/pysam 等 | 调用的是哪个模块、该模块依赖；改入口或在新环境补齐 | 目标模块导入及实际受影响步骤 |
| 成对分类/BAM 记录不闭合 | read ID 命名空间、主记录、排序、逐 read 过滤、RG/LB 与片段逻辑 | 已知答案边界测试和真实独立计数 |
| 峰数比注释行数多 | 支架无 TxDb 特征、重复坐标连接、ID 映射与过滤 | 全峰 ID 一一保留，Unannotated 有原因 |
| 低 MAPQ/低宿主有效率 | 物种/组装、read 长度、接头、参考、竞争序列、原日志；按需来源排查 | 小型代表性真实数据、各终止类别和限制 |
| FastQC FAIL/高重复 | 区分模块警报、技术失败和实验设计；结合靶标/片段/对照 | raw/clean 比较，适用指标和实验解释 |
| TSS 文件缺失、短基因/Seqinfo warning | 当前区域模式、组装/序列名、bin/长度和实际跳过语义 | 正确区域、矩阵范围与保留/跳过数量 |
| Excel 基因名变日期/小 P 变零 | 导出类型、原值字符串、JSON/浮点格式 | 实际重读 XLSX，和完整 TSV 对比 |
| HTML 很大、Top 10 被当全量 | 是否内嵌原图/全表、完整附件是否存在 | 主页面/全表/原图同行，搜索/末页/导出实测 |
| HTML 附件打不开 | 相对路径、复制完整性、file/HTTP 模式、浏览器错误 | 在最终使用方式下实测，不能只用文件存在检查 |
| IGV 启动/截图失败 | 平台包、Java、显示环境、参考和 BAI；独立截图测试 | 实际截图、选区/纵轴与会话依赖 |
| 任务看似仍 running | 实际 PID/启动时间/命令、锁持有者、机器重启和日志 | 确认无重复执行后标记中断并受控续跑 |
| 磁盘不足/排序慢 | 并行数、每线程内存、heap、临时目录和真实 I/O | 新资源预算、小批实测，不用百分比承诺耗时 |
| 外盘掉线/复制错误 | 停止依赖读入，记录异常时间，检查挂载/设备/线材/供电及日志 | 从来源清单核对副本；程序路径修复不证明硬件根因消除 |
| 清单失败/报告目录混入新文件 | 包是否冻结、是否后来添加另一个报告、实际来源关系 | 另建版本与新清单，不修改旧清单掩盖差异 |

无法定位根因时保留“已有防护、根因未确认”，提供最小复现与受影响范围。不要把工具 warning 的关键词数量统计成故障次数，也不要无限自动重试或结束所有 Python 进程。

## 16. 失败恢复、复用和性能记录

这些是供 AI 实现的调度要求；本文没有附带可直接运行的队列/恢复程序。

### 16.1 状态与执行

可以定义 draft、queued、running、paused、failed、interrupted、cancelled、completed。每个步骤保存开始/结束、退出码、尝试编号、命令与产物；后台 stdin/stdout/stderr 明确配置，避免继承失效标准流。取消应针对对应进程/进程组，先保存状态，避免误杀无关进程。

步骤在独立临时目录产生结果，核对完成后原子发布。`.partial`、零字节输出或仅存在目录都不代表成功。检查点可设在 QC 后和比对/过滤后，查看产物与实验限制，再释放后续步骤。

### 16.2 复用依据

步骤指纹至少覆盖：

- 输入实际内容哈希、参数和样本关系。
- 该步骤实际代码/依赖代码、环境包集合与外部工具身份。
- FASTA/索引/区域/注释/数据库的内容身份与版本。
- 上游产物清单及输出完整性。

文件路径和样本名相同不能证明内容相同。大文件哈希缓存需要可靠的文件身份/变更防护；路径不变、内容变化要失效。同一运行中发现参考漂移应拒绝继续；更新参考另建版本。

修改上游方法时重算受影响下游；纯 HTML/CSS 改动应只影响报告层，不能因此无范围重跑比对。内部版补成交付版时若缺 GO/KEGG/motif 等科学结果，需要明确补算，不能宣称纯版式转换。

旧锁 PID 或状态可能残留；确认实际进程后再恢复。已完成/取消运行不直接覆盖，另建版本。硬链接复用产物不是独立备份，禁止原地编辑共享文件；采用临时生成后替换。

### 16.3 耗时与资源

分开记录下载、科学计算、打包、浏览器检查和等待。管道尽可能记录每个子命令的时间/退出码，采样进程树资源与磁盘变化；采样 RSS 不能当精确内存峰值。

估时匹配机器、参考、输入量和参数，给出依据与范围；只有少数历史批次时不承诺固定提速倍数或完成时间。进度条可以是步骤比例，但不能称为已完成计算时间比例。

## 17. 归档、迁移、清理与维护

### 17.1 归档和恢复

归档包含原始输入/来源清单、样本/参数、代码/环境锁、参考来源/身份、正式结果/完整报告、日志/验收和实际外部工具版本。需要续跑、重绘或未比对排查时，另外保留对应过程 BAM、矩阵等；最终宿主 BAM 不能替代所有中间数据。

保存复制前已有哈希清单，在副本逐项核对，实际打开报告/附件和会话。记录复制位置、时间、验证者、检查范围与失败项；“使用者确认已复制”“AI 实际校验”“未核验”分别填写。未经副本核验，不把本机文件删除当成归档完成。

换电脑时，新根目录部署环境并核验参考，恢复数据/结果和原记录；另建迁移映射与新验证记录。原运行快照中的旧路径保留其历史含义，不批量改写来伪装旧结果在新位置生成。通过新机代表性验证后再决定旧部署保留策略。

### 17.2 分级清理

第一层在交付验收后评估 scratch/临时文件；仍需续跑、复用、排查或重绘的中间产物保留。第二层在外部归档并校验后，按明确对象评估本机该项目输入/结果/中间数据。两层都先确认无活动任务和引用关系。

默认保护环境、Conda、参考、代码、配置/锁、记录和必要源表。解包缓存可能与环境有链接依赖，不直接删除整个目录。逻辑文件总字节、去重 inode 体积和 APFS 实际可释放空间不同，不承诺按表面大小等量释放。

清理前应有明确对象、理由、归档证据与当次授权。本指南和“分析完成”状态不自动执行删除。

### 17.3 更新和文档维护

先写范围、保存旧文件/实际锁与状态，候选前缀验证单模块；保留旧版本，按影响补软件回归和必要真实验证。不要无范围 `conda update --all`，也不要随意重命名环境前缀完成切换。

每次更新记录问题、证据、修复、实际验证、文档变化和剩余限制。新问题按症状留原文，后续成功不覆盖旧失败；“根因已定位”“增加防护”“暂未复现”分别写。对外文档只展示可分享的脱敏经验，不能直接复制包含真实样本/私人路径的内部记录。

## 18. 本地建成后的验收与交接

| 层级 | 通过要求 | 不能替代什么 |
|---|---|---|
| 环境就绪 | 匹配平台、工具可运行、小型实际操作、来源/锁齐全 | 真实样本分析正确性 |
| 合成机制 | 成对分类闭合、已知轨道/FRiP、正反例、恢复/报告边界 | 真实实验质量、校准有效或来源确认 |
| 代表性真实验证 | 获准输入和匹配参考，全链路产物/数值核对 | 未测试平台、方法或全部正式批次 |
| 正式运行 | 批次参数/证据、完整结果、角色/零值/模块条件正确 | 实验设计缺失所需的解释依据 |
| 报告交付 | 全表/文件/下载/原图/会话与副本清单实测 | 没有执行过的离线/浏览器场景 |

最终交接至少包含工作目录和入口、平台/环境锁、样本与待确认项、参考清单、运行配置、代码版本、实际启用模块、完整包/私有记录位置、验收与剩余问题、归档和保留范围。

对外可以如实说明这条路线有真实本机实践价值，同时标明自己的新增实现与验证范围。让后续 AI 能从记录继续，而不是依靠对话记忆重新猜测。

更多来源见 [官方资料索引](SOURCES.md)；交给 AI 时使用 [执行入口](AI_START_HERE.md)。
