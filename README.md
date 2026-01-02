# Repeat and haplotype aware error correction in nanopore sequencing reads with DeChat

[![BioConda](https://img.shields.io/conda/dn/bioconda/DeChat?label=bioconda%20downloads)](https://anaconda.org/bioconda/DeChat)
![BioConda](https://img.shields.io/conda/vn/bioconda/DeChat?label=bioconda)
![License](https://img.shields.io/github/license/LuoGroup2023/DeChat)
![Release](https://img.shields.io/github/v/release/LuoGroup2023/DeChat?sort=semver)
![Downloads](https://img.shields.io/github/downloads/LuoGroup2023/DeChat/total)
![Stars](https://img.shields.io/github/stars/LuoGroup2023/DeChat?style=social)

## Fork notice / 分叉说明

本仓库为 LuoGroup2023/DeChat 的非官方分叉，包含若干构建与运行相关修复与改进。
This repository is an unofficial fork of LuoGroup2023/DeChat and includes fixes/improvements related to build and runtime behavior.

上游版本存在一个已知问题：当用户通过 -t 指定线程数时，correct_round1 阶段可能不会遵循该设置，而是占用全部可用 CPU 核心。相关修复已作为 PR 提交至上游，但目前尚未被合并:https://github.com/LuoGroup2023/DeChat/pull/13
\
The upstream version has a known issue: when users specify -t, the correct_round1 stage may ignore this setting and use all available CPU cores. A fix has been submitted upstream but has not been merged yet:https://github.com/LuoGroup2023/DeChat/pull/13

我通过修改使用的第三方库，向 merge.cpp 和 kff_io.cpp 中添加 `#include <cstdint>`，使该项目能够支持 gcc13/g++13 编译。\
I adjusted the third-party library in use and added `#include <cstdint>` to merge.cpp and kff_io.cpp, enabling compilation with gcc 13 / g++ 13.

我在本分叉中引入了 SIMDe 兼容层，使 DeChat 能够在 aarch64 Linux 设备上完成编译并运行。但由于缺乏足够的样本验证，我无法保证在 aarch64 平台上的运行结果与其他平台完全一致或其结果准确性。\
I introduced SIMDe (SIMD everywhere) compatibility in this fork so that DeChat can be built and run on aarch64 Linux devices. However, due to limited samples, I cannot guarantee that results on aarch64 are fully identical to those on other platforms or that the output accuracy is fully verified.

此外，我对本 fork 的可用构建依赖版本范围进行了验证；请以 compilation.yaml 中记录的依赖与版本约束为准。\
In addition, I validated the workable dependency/version range for building this fork; please refer to compilation.yaml for the authoritative dependency list and version constraints.

我已知该项目仍存在一个 bug：其最大只能使用 64 线程，但我暂时没时间修复。\
I am aware of an existing bug that the project supports at most 64 threads, but I have not had time to fix it yet.
## Description

Error correction is the canonical first step in long-read sequencing data analysis. Nanopore R10 reads have error rates below 2\%. we introduce DeChat, a novel approach specifically designed for Nanopore R10 reads.DeChat enables repeat- and haplotype-aware error correction, leveraging the strengths of both de Bruijn graphs and variant-aware multiple sequence alignment to create a synergistic approach. This approach avoids read overcorrection, ensuring that variants in repeats and haplotypes are preserved while sequencing errors are accurately corrected.

DeChat can use HIFi or NGS to correct ONT now

Dechat is implemented with C++.

## Installation and dependencies / 安装与依赖

可复现的构建依赖以 `compilation.yaml` 为准；此处仅列出关键依赖及已验证可用的版本范围。  
For a reproducible build, please refer to `compilation.yaml`. The list below highlights the key dependencies and the tested version ranges.

- GCC/GXX: **>= 9.5** and **< 14**  
已知在 GXX 14 下无法通过编译，原因尚待进一步定位。 \
Building with GXX 14 is known to fail; the root cause is still under investigation.
- CMake: **>= 3.2** 
- zlib
- Boost: **>= 1.67** and **< 1.73**  
已知在 Boost >= 1.73.0 下无法通过编译，原因尚待进一步定位。 \
Building with Boost >= 1.73.0 is known to fail; the root cause is still under investigation.

部分系统发行版的软件源可能不提供较旧的 Boost；因此建议使用 Conda 环境进行构建 \
Some OS package managers may not ship older Boost versions; using a Conda environment is recommended.

### 1) Install from Conda / 通过 Conda 安装

> ⚠️ 说明（关于 Conda/Bioconda 包）/ Note (about Conda/Bioconda package)
>
> 上游版本存在一个已知问题：当用户通过 `-t` 指定线程数时，`correct_round1` 阶段可能不会遵循该设置，而是占用全部可用 CPU 核心。
> 相关修复已作为 PR 提交至上游，但目前尚未被合并：
> https://github.com/LuoGroup2023/DeChat/pull/13
>
> The upstream version has a known issue: when users specify `-t`, the `correct_round1`
> stage may ignore this setting and use all available CPU cores.
> A fix has been submitted upstream but has not been merged yet:
> https://github.com/LuoGroup2023/DeChat/pull/13


如果你仅希望快速试用，可尝试 Conda 安装（但不包含上述修复）：  
If you only want a quick try, you may install via Conda (the fix above is not included):

```bash
conda create -n dechat
conda activate dechat
conda install -c bioconda dechat
```
#### 2) Install from source / 从源码安装（推荐） 
```bash
git clone https://github.com/qiyuanhuakai/DeChat.git
conda env create -n dechat --file compilation.yaml
conda activate dechat
```
或使用 mamba（解析依赖更快）：\
Or use mamba (faster environment solving):

```bash
git clone https://github.com/qiyuanhuakai/DeChat.git
mamba create -n dechat -f compilation.yaml
mamba activate dechat
```
## Build & Run / 编译与运行
建议使用 Ninja 生成器进行构建，可略微提升编译速度。\
Ninja is recommended for a slightly faster build.

```bash
cd DeChat
rm -rf build 
mkdir build
cmake -S . -B build -G Ninja
cmake --build build
./bin/dechat
```

>**关于编译警告 / About warnings**
>
>编译过程中会出现极多 warning（尤其是使用 GCC 13 时）。如需减少，可考虑使用 GCC 11 或更低版本，或在配置时添加：\
You may see many warnings during compilation (especially with GCC 13).To reduce them, consider using GCC 11 or older, or add the following CMake options:
>
>`-Wno-dev -DCMAKE_C_FLAGS="-w" -DCMAKE_CXX_FLAGS="-w"`



The input read file is only required and the format should be FASTA/FASTQ (can be compressed with gzip). Other parameters are optional.
Please run `dechat` to get details of optional arguments. 

```
Repeat and haplotype aware error correction in nanopore sequencing reads with DeChat
Usage: dechat [options] -o <output> -t <thread>  -i <reads> <...>
Options:
  Input/Output:
       -o STR       prefix of output files [(null)]
                   The output for the stage 1 of correction is "recorrected.fa", 
                    The final corrected file is "file name".ec.fa;
       -t INT       number of threads [1]
       -h           show help information
       -v --version show version number
       -i           input reads file
       -k INT       k-mer length (must be <64) [21]
  Error correction stage 1 (dBG):
       -r1           set the maximal abundance threshold for a k-mer in dBG [2]
       -d           input reads file for building dBG (Default use input ONT reads) 
  Error correction stage 2 (MSA):
       -r            round of correction in alignment [3]
       -e            maximum allowed error rate used for filtering overlaps [0.04]     
```

## Examples

The example folder contains test data, including the 10X depth sim-ont10.4 data of Escherichia coli diploid and its corresponding reference sequence. The running method is as follows:
```
cd example
dechat -i reads.fa.gz -o reads -t 8
```
### Using HIFi or NGS to correct ONT
```
cd example
dechat -i reads.fa.gz -o reads -t 8 -d HiFi-reads.fq.gz/NGS-reads.fq.gz
```


