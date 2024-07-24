# CUT-TAG分析流程以及详细解读

---

## 0.进入环境 

```
source activate CUT-TAG
```



---

## 1. Fastp: 数据质量控制

```bash
fastp -i sample1.fastq.gz -I sample2.fastq.gz -o sample1.clean.fastq.gz -O sample2.clean.fastq.gz -j sample.json
```

- `-i`: 输入的FASTQ文件（正向读取）。
- `-I`: 输入的FASTQ文件（反向读取）。
- `-o`: 输出的清理后的FASTQ文件（正向读取）。
- `-O`: 输出的清理后的FASTQ文件（反向读取）。
- `-j`: 质量控制统计的JSON报告文件。

__统计所有数据原始测序量，Q20，Q30，及clean后的各项数据__

---

## 2. Bowtie2比对和Samtools处理

```bash
bowtie2 -p 30 --very-sensitive -X 2000 -x $bowtie2_index -1 sample_1.fq.gz -2 sample_2.fq.gz | samtools sort -O bam -@ 5 -o sample.raw.bam
samtools index sample.raw.bam
samtools flagstat sample.raw.bam
```

- `-p 30`: 使用30个线程进行比对。
- `--very-sensitive`: 使用非常敏感的比对模式。
- `-X 2000`: 配对读取的最大插入片段长度为2000 bp。
- `-x`: 参考索引前缀。
- `-1`, `-2`: 配对端读取的输入FASTQ文件。
- `samtools sort`: 对SAM/BAM文件进行排序。
- `-@ 5`: 使用5个线程进行排序。
- `samtools index`: 为BAM文件创建索引。
- `samtools flagstat`: 报告BAM文件的统计信息。


**此处统计数据比对率，去重前数据量**

---

## 3. 使用Sambamba标记重复

```bash
sambamba1 markdup --overflow-list-size 600000 --tmpdir='./' -r sample.raw.bam sample.rmdup.bam
samtools index sample.rmdup.bam
samtools flagstat sample.rmdup.state
```

- `--overflow-list-size 600000`: 处理溢出记录的列表大小。
- `--tmpdir='./'`: 临时文件目录。
- `-r`: 标记重复并创建新的BAM文件（`sample.rmdup.bam`）。
- `samtools index`: 为去重后的BAM文件创建索引。
- `samtools flagstat`: 报告去重后的BAM文件的统计信息。

**此处统计去重后数据量，根据raw.bam数据量计算重复率**

---

## 4. 过滤和排序BAM文件

```bash
samtools view -h -f 2 -q 30 sample.rmdup.bam | grep -v chrM | samtools sort -O bam -@ 5 -o sample.last.bam
samtools index sample.last.bam
samtools flagstat sample.last.bam > sample.last.state
```

- `-h`: 包括头信息。
- `-f 2`: 仅包含正确配对的读取。
- `-q 30`: 过滤掉映射质量低于30的读取。
- `grep -v chrM`: 排除映射到线粒体染色体（chrM）的读取。
- `samtools sort`: 对过滤后的BAM文件进行排序。
- `-@ 5`: 使用5个线程进行排序。
- `samtools index`: 为最终排序的BAM文件创建索引。
- `samtools flagstat`: 报告最终BAM文件的统计信息。

**此处统计fragments数，相当于过滤后剩余多少对reads**

---

## 5. 生成BigWig文件

```bash
bamCoverage --bam sample.last.bam -o sample.bw --binSize 10 --extendReads --normalizeUsing RPKM
```

- `--bam`: 输入的BAM文件。
- `-o`: 输出的BigWig文件。
- `--binSize 10`: BigWig文件中分割的bin的大小。
- `--extendReads`: 扩展读取用于计算覆盖度。
- `--normalizeUsing RPKM`: 每个bins中read counts的校正单位，使用RPKM（每百万读取的转录本每千碱基对）进行归一化。

**生成bw文件，要和peak文件共同可视化**

---

## 6. 峰值调用

**STARR-seq:**

```bash
macs2 callpeak -t sample -c input.last.bam -g 2728702538 -p 0.01 -f BAMPE -B --SPMR --keep-dup=all -n sample --outdir ./ 
```

**CHIP/ CUT&TAG/ATAC:**

```bash
macs2 callpeak -t sample -g 2728702538 -p 0.01 --nomodel --shift -75 --extsize 150 -B --SPMR --keep-dup all --call-summits -n sample --outdir ./ 
```

- `-t`: 输入的BAM文件（实验样本）。
- `-c`: 对照组的BAM文件（如有）。
- `-g 2728702538`: 有效基因组的大小（通常为基因组的总长度，单位为碱基对）。
- `-p 0.01`: 显著性水平（p值阈值）。
- `-f BAMPE`: 输入格式为配对端BAM。
- `--nomodel`: 不建立模型。
- `--shift -75`: 读取向左移动75 bp。
- `--extsize 150`: 扩展读取到150 bp。
- `-B`: 生成峰值文件。是否输出bedgraph格式的文件。以bedGraph格式存放fragment pileup, control lambda, -log10pvalue和log10qvalue。
- `--SPMR`: 将bedGraph文件中的读数转化为每百万reads的比率。
- `--keep-dup all`: 保留所有重复的读取。
- `--call-summits`: MACS利用此参数重新分析信号谱，解析每个peak中包含的subpeak。对相似的结合推荐使用此参数，当使用此参数时，输出的subpeak会有相同的peak边界，不同的绩点和peak summit poisitions.
- `-n sample`: 输出文件的前缀名称。
- `--outdir ./`: 输出目录。

*-c 放input数据 --keep-dup all 前一步已经去重了这里要保证不去重*

**获得narrowpeak文件后与bw一起放入IGV，看call peak对不对**

---
