# 利用stringtie进行表达量分析
生成参考转录本`.gtf`文件，可以使用`.gff`注释文件  
-p 4：每个 StringTie 任务使用 4 个 CPU 线程  
-G ../../annotation/rn6.gff：参考注释文件（大鼠 rn6 基因组）  
-l {1}：设置样本标签为 {1}（占位符）  
{1}.sort.bam：输入文件（排序后的 BAM 文件）  
-o ../assembly/{1}.gtf：输出 GTF 文件路径  

```bash
cd ~/project/rat/output
mkdir assembly

cd align
parallel -j 4 "
    stringtie -p 4 -G ../../annotation/rn6.gff -l {1} {1}.sort.bam -o ../assembly/{1}.gtf
" ::: $(ls *.sort.bam | perl -p -e 's/\.sort\.bam$//')
```

# 查看新生成的.gtf文件

```bash
cd ~/project/rat/output/assembly
cat SRR2190795.gtf | head -n 4
```

输出结果如下，可以看到使用StringTie处理后的注释文件与之前不一样了，在第九列里面出现了更多的描述信息，包括比对上的样本名，每个碱基的覆盖深度的平均数cov，表达量的标准化数值TPM和FPKM等等  

```
# stringtie -p 4 -G ../../annotation/rn6.gff -l SRR2190795 SRR2190795.sort.bam -o ../assembly/SRR2190795.gtf
# StringTie version 3.0.0
1       StringTie       transcript      237620  249724  1000    -       .       gene_id "SRR2190795.1"; transcript_id "SRR2190795.1.1"; cov "1.611305"; FPKM "114.012878"; TPM "140.421066";
1       StringTie       exon    237620  237789  1000    -       .       gene_id "SRR2190795.1"; transcript_id "SRR2190795.1.1"; exon_number "1"; cov "1.317647";
```

# 合并.gtf文件

```bash
cd ~/project/rat/output/assembly

# 将生成 .gtf 文件的文件名放到文件中
ls *.gtf > mergelist.txt

##### 合并 ######
# --merge 合并模式
# -p 线程数
# -G 参考基因组的
stringtie --merge -p 8 -G ../../annotation/rn6.gff -o merged.gtf mergelist.txt 
```
参数--merge 为转录本合并模式。 在合并模式下，stringtie将所有样品的GTF/GFF文件列表作为输入，并将这些转录本合并/组装成非冗余的转录本集合。这种模式被用于新的差异分析流程中，用以生成一个跨多个RNA-Seq样品的全局的、统一的转录本

# 估计转录本的丰度
转录本的丰度，就是指某个特定转录本在某个生物样本中的有多少  

```bash
cd ~/project/rat/output
mkdir abundance

cd align

# -e：关键参数，表示仅估算与 -G 选项提供的转录本匹配的丰度，不组装新转录本
# -B：生成 Ballgown 软件所需的输入表文件（.ctab），用于下游差异表达分析
# -l {1}：设置输出转录本名称的前缀为样本名
parallel -j 4 "
    mkdir -p ../abundance/{1}
    stringtie -e -B -p 4 -G ../assembly/merged.gtf -l {1} {1}.sort.bam -o ../abundance/{1}/{1}.gtf
" ::: $(ls *.sort.bam | perl -p -e 's/\.sort\.bam$//')
```

每个样本的输出目录里都有以下文件：
```
e_data.ctab
e2t.ctab
i_data.ctab
i2t.ctab
SRR2190795.gtf
t_data.ctab
```

# 对比原始的注释文件，查看有哪些新发现的转录本

```bash
# 没写出具体流程
gffcompare 
```

# 转化为表达量的矩阵
下载一个转换脚本，这个python脚本可以把之前生成的.gtf文件转换为表达量矩阵  
脚本使用的是Python2，在Ubuntu（不是Ubuntu-24.04）中下载Python2，将整个文件夹复制到Ubuntu的目录中，执行完命令后将`matrix`中的文件复制回来  

```bash
cd ~/project/rat/script

wget https://ccb.jhu.edu/software/stringtie/dl/prepDE.py

# 注意这个脚本是使用python2
python2 prepDE.py --help

cd ~/project/rat/output/
mkdir matrix

# 开始进行转换
# -i:输入文件；-g：输出文件 gene_count_matrix.csv 的位置；-t：输出文件 transcript_count_matrix.csv 的位置；
# -l：平均read长度
python2 ~/project/rat/script/prepDE.py \
   -i ./abundance \
   -g ./matrix/gene_count_matrix.csv \
   -t ./matrix/transcript_count_matrix.csv \
   -l 100
```

# 整理一下文件夹，使用tree命令查看我们的目录结构

```bash
cd ~/project/rat

# -d 参数表示只显示文件夹
tree -d
```