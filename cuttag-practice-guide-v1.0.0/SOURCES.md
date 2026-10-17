# 官方资料与方法核对入口

整理日期：2026-10-17。固定 release 链接用于描述实践基线，stable/release 页面会更新；执行时核对自己实际安装版本的帮助/文档并保存版本。不根据网页“最新”静默升级环境。

本指南中的参数、分类顺序、轨道算法、报告规则和恢复要求包含作者实践选择；来源用于核对工具语义，不表示每个实践默认值都是官方 CUT&Tag 合格标准。

| 主题 | 一手资料 | 用于核对 |
|---|---|---|
| Miniforge | [官方仓库](https://github.com/conda-forge/miniforge)、[固定版本](https://github.com/conda-forge/miniforge/releases/tag/26.7.2-0) | 架构安装器、校验、批量安装 |
| Conda | [管理环境](https://docs.conda.io/projects/conda/en/stable/user-guide/tasks/manage-environments.html) | 前缀、需求导出与同平台 explicit 恢复 |
| Bioconda | [平台和配置](https://bioconda.github.io/) | channel、strict 与平台适用范围 |
| Windows/WSL | [微软 WSL 安装](https://learn.microsoft.com/en-us/windows/wsl/install) | Linux 子系统的部署前提 |
| FastQC | [项目主页](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) | 质量模块、报告与实际版本 |
| Cutadapt | [使用指南](https://cutadapt.readthedocs.io/en/stable/guide.html) | 双端修剪、pair filter、JSON/参数语义 |
| Bowtie2 | [官方手册](https://bowtie-bio.sourceforge.net/bowtie2/manual.shtml) | dovetail、mixed/discordant、插入范围、RG |
| SAM/BAM | [Samtools/HTSlib](https://www.htslib.org/doc/samtools.html)、[格式规范](https://samtools.github.io/hts-specs/) | 标记、排序、索引、主比对与格式 |
| Picard | [命令概览](https://broadinstitute.github.io/picard/command-line-overview.html) | MarkDuplicates、读名/RG/LB、重复指标 |
| MACS3 | [callpeak](https://macs3-project.github.io/MACS/docs/callpeak.html) | BAMPE、重复政策、broad、effective genome size |
| bedtools | [intersect](https://bedtools.readthedocs.io/en/stable/content/tools/intersect.html)、[genomecov](https://bedtools.readthedocs.io/en/latest/content/tools/genomecov.html) | 重叠只计一次、片段区间覆盖 |
| GENCODE | [人源 v49](https://www.gencodegenes.org/human/release_49.html) | GRCh38 primary FASTA 与匹配 GTF |
| blacklist | [Boyle-Lab 仓库](https://github.com/Boyle-Lab/Blacklist)、[固定 hg38 v2](https://raw.githubusercontent.com/Boyle-Lab/Blacklist/61a04d2c5e49341d76735d485c61f0d1177d08a8/lists/hg38-blacklist.v2.bed.gz) | 组装适用性、原始排除区间 |
| deepTools | [computeMatrix](https://deeptools.readthedocs.io/en/latest/content/tools/computeMatrix.html)、[相关性](https://deeptools.readthedocs.io/en/latest/content/tools/plotCorrelation.html)、[PCA](https://deeptools.readthedocs.io/en/latest/content/tools/plotPCA.html) | 矩阵、区域/缺失政策与探索性图 |
| Bioconductor | [安装与版本匹配](https://bioconductor.org/install/)、[3.22 ChIPseeker](https://bioconductor.org/packages/3.22/bioc/html/ChIPseeker.html)、[txdbmaker](https://bioconductor.org/packages/txdbmaker/) | R/Bioc 组合、同版 GTF→TxDb 与峰注释 |
| clusterProfiler | [官方包入口](https://bioconductor.org/packages/clusterProfiler/) | universe、ID、富集方法及实际版本手册 |
| KEGG | [REST 说明](https://www.kegg.jp/kegg/rest/keggapi.html)、[数据使用说明](https://www.kegg.jp/kegg/legal.html) | 数据来源、接口与使用条件 |
| HOMER | [官方 motif 入口](http://homer.ucsd.edu/homer/motif/) | 自定义参考/背景、known motif 与实际版本帮助 |
| MultiQC | [官方文档](https://docs.seqera.io/multiqc/) | 工具日志聚合与报告范围 |
| IGV | [下载](https://igv.org/doc/desktop/DownloadPage/)、[batch](https://igv.org/doc/desktop/UserGuide/tools/batch/)、[命令行](https://igv.org/doc/desktop/UserGuide/advanced/command_line/) | 平台/Java、截图/会话与批处理 |
| BLAST+ | [NCBI BLAST Help](https://blast.ncbi.nlm.nih.gov/doc/blast-help/)、[数据库](https://blast.ncbi.nlm.nih.gov/doc/blast-help/blastdatabases.html)、[公共服务规则](https://blast.ncbi.nlm.nih.gov/doc/blast-help/developerinfo.html) | 核酸库范围、输出/分类、外部搜索前提 |
| Excel 导出替代路线 | [openpyxl 官方文档](https://openpyxl.readthedocs.io/en/stable/) | 文本/数值类型、写入和实际重读检查 |

网页暂不可达时，先保留错误，使用已安装工具的 `--help`、R 包手册或上游固定版本文档核对。没有取得的内容不凭记忆补成已核验事实；数据库和软件也不能由链接存在推断已经下载。
