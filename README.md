# Genome annotation pipeline (EVM)

> ***By Yang Mei***
> 
> ***Institution: Zhejiang University***
> 
>  ***Email: meiyang12@zju.edu.cn***
>  
>  ***Cite: Yang Mei, Dong Jing, Shenyang Tang, Xi Chen, Hao Chen, Haonan Duanmu, Yuyang Cong, Mengyao Chen, Xinhai Ye, Hang Zhou, Kang He, Fei Li, InsectBase 2.0: a comprehensive gene resource for insects, Nucleic Acids Research, Volume 50, Issue D1, 7 January 2022, Pages D1040–D1045, https://doi.org/10.1093/nar/gkab1090.***


------

### 1. Prerequisites

#### 1.1 Software

1. BUSCO (https://busco.ezlab.org/)
2. RepeatMasker, RepeatModeler (http://www.repeatmasker.org/)
3. HISAT2 (http://daehwankimlab.github.io/hisat2/)
4. StringTie (http://ccb.jhu.edu/software/stringtie/)
5. TransDecoder (https://github.com/TransDecoder/TransDecoder)
6. BRAKER (https://github.com/Gaius-Augustus/BRAKER)
   + AUGUSTUS (https://github.com/Gaius-Augustus/Augustus)
   + GeneMark (http://topaz.gatech.edu/GeneMark/)
   + BAMTOOLS (https://github.com/pezmaster31/bamtools)
   + SAMTOOLS (http://www.htslib.org/)
   + ProtHint (https://github.com/gatech-genemark/ProtHint)
   + DIAMOND (http://github.com/bbuchfink/diamond/)
7. NCBI BLAST+ (https://blast.ncbi.nlm.nih.gov/Blast.cgi)
8. miniprot (https://github.com/lh3/miniprot)
9. EVidenceModeler (https://github.com/EVidenceModeler/EVidenceModeler/wiki)
10. PASA (https://github.com/PASApipeline/PASApipeline/wiki)
    + Blat (http://hgdownload.soe.ucsc.edu/admin/exe/linux.x86_64/blat/blat)
    + fasta (http://faculty.virginia.edu/wrpearson/fasta/fasta36/fasta-36.3.8g.tar.gz)
11. gffread (https://github.com/gpertea/gffread)

#### 1.2 DataSet

1. RNA-seq (Optional) (https://www.ncbi.nlm.nih.gov/sra/)
2. Homology protein (eg., OrthoDB (https://v100.orthodb.org/))
   + Fungi: https://v100.orthodb.org/download/odb10_fungi_fasta.tar.gz
   + Metazoa: https://v100.orthodb.org/download/odb10_metazoa_fasta.tar.gz
     + Arthropoda: https://v100.orthodb.org/download/odb10_arthropoda_fasta.tar.gz
     + Vertebrata: https://v100.orthodb.org/download/odb10_vertebrata_fasta.tar.gz
   + Protozoa: https://v100.orthodb.org/download/odb10_protozoa_fasta.tar.gz
   + Viridiplantae: https://v100.orthodb.org/download/odb10_plants_fasta.tar.gz

------

### 2. Genome assessment

> #### *BUSCOv4*
>
> + genome.fa
> + insecta_odb10

```shell
busco --cpu 28 \
	-l /gpfs/home/meiyang/opt/insecta_odb10 \
	--config /gpfs/home/meiyang/opt/busco-4.0.5/config/config.ini \
	--mode genome --force -o busco \
	--i genome.fa \
	--offline
```

------

### 3. Repeat annotation and genome mask

> #### *RepeatModelerv2 & RepeatMasker*
>
> + genome.fa

```bash
famdb.py -i Libraries/RepeatMaskerLib.h5 families -f embl  -a -d Insecta  > Insecta_ad.embl

util/buildRMLibFromEMBL.pl Insecta_ad.embl > Insecta_ad.fa

# RepeatModeler
mkdir 01_RepeatModeler
BuildDatabase -name GDB -engine ncbi ../genome.fa > BuildDatabase.log
RepeatModeler -engine ncbi -pa 28 -database GDB -LTRStruct > RepeatModele.log
cd ../

# RepeatMasker
mkdir 02_RepeatMasker
cat 01_RepeatModeler/GDB-families.fa Insecta_ad.fa > repeat_db.fa
```

```shell
RepeatMasker -xsmall -gff -html -lib repeat_db.fa -pa 28 genome.fa > RepeatMasker.log
```

> genome.fa.masked

------

### 4. Gene prediction

#### 4.1 Ab initio gene prediction

> #### *BRAKERv2*
>
> + masked geome (genome.fa)
> + homology protein (OrthoDB), proetin.fasta
> + --species=<species_name>
> + --min_contig, less than genome N50

```shell
braker.pl --species=Sfru \
	--genome=genome.fa \
	--prot_seq=protein.fasta \
	--softmasking --gff3 --cores=16 \
	--workingdir=ab_initio \
	--min_contig=4000

mv augustus.hints.gff3 gene_predictions.gff3
```

> **gene_predictions.gff3**

#### 4.2 RNA-seq based gene prediction

> #### 1. *HISAT2 & StringTie*
>
> + masked geome (genome.fa)
>+ transcriptome
> 

```shell
# genome index
hisat2-build -p 28 genome.fa genome

# -------------------------------------------------------------------------------------- #
# single end
hisat2 -p 28 -x genome --dta -U reads.fq | samtools sort -@ 28 > reads.bam 
# paired end
hisat2 -p 28 -x genome --dta -1 reads_1.fq -2 reads_2.fq | samtools sort -@ 28 > reads.bam
...
```

```shell
# batch running
# single end
single_list = './single.txt'
for run in `cat $single_list`
do
	hisat2 -p 28 -x genome --dta -U ${run}.fq | samtools sort -@ 28 > ${run}.bam
done

# -------------------------------------------------------------------------------------- #
# paired end
paired_list = './paired.txt'
for run in `cat $paired_list`
do
	hisat2 -p 28 -x genome --dta -1 ${run}_1.fq -2 ${run}_2.fq | samtools sort -@ 28 > ${run}.bam
done
```

```shell
# merge gtf
samtools merge -@ 28 merged.bam `ls *bam`
stringtie -p 28 -o stringtie.gtf merged.bam
```

> stringtie.gtf

> #### 2. *TransDecoder*
>
> + masked geome (genome.fa)
> + stringtie.gtf

```shell
util/gtf_genome_to_cdna_fasta.pl stringtie.gtf genome.fa > transcripts.fasta

util/gtf_to_alignment_gff3.pl stringtie.gtf > transcripts.gff3

TransDecoder.LongOrfs -t transcripts.fasta

# -------------------------------------------------------------------------------------- #
# homology search
blastp -query transdecoder_dir/longest_orfs.pep -db uniprot_sprot.fasta  -max_target_seqs 1 -outfmt 6 -evalue 1e-5 -num_threads 28 > blastp.outfmt6
hmmscan --cpu 28 --domtblout pfam.domtblout Pfam-A.hmm transdecoder_dir/longest_orfs.pep
# -------------------------------------------------------------------------------------- #

TransDecoder.Predict -t transcripts.fasta --retain_pfam_hits pfam.domtblout --retain_blastp_hits blastp.outfmt6

util/cdna_alignment_orf_to_genome_orf.pl transcripts.fasta.transdecoder.gff3 transcripts.gff3 transcripts.fasta > transcripts.fasta.transdecoder.genome.gff3

mv transcripts.fasta.transdecoder.genome.gff3 transcript_alignments.gff3
```

> **transcript_alignments.gff3**

#### 4.3 Homology-based gene prediction

> #### *miniprot*
>
> + masked geome (genome.fa.masked)
> + homology protein (OrthoDB), proetin.fasta

```shell
miniprot -t28 -d genome.mpi genome.fa.masked 

miniprot -It28 --gff genome.mpi protein.fasta > miniprot.gff3

python miniprot.py miniprot.gff3 > protein_alignments.gff3

```

> **protein_alignments.gff3**



------

### 5 EVidenceModeler (EVM)

#### 5.1 Preparing Inputs

+ ##### masked geome (genome.fa)

+ ##### weights.txt

```shell
PROTEIN	miniprot	5
ABINITIO_PREDICTION	AUGUSTUS	4
OTHER_PREDICTION	transdecoder	10
```


+ ##### GFF3 file

  + gene_predictions.gff3
  + protein_alignments.gff3
  + transcript_alignments.gff3

+ #####  Check the gff3 file (Optional)

  ```shell
  gff3_gene_prediction_file_validator.pl your.gff3
  ```

  

#### 5.2 Run

```shell
EVidenceModeler \
	--sample_id speceis \
	--genome genome.fa \
	--weights weights.txt  \
	--gene_predictions gff/gene_predictions.gff3 \
	--protein_alignments gff/protein_alignments.gff3 \
	--transcript_alignments gff/transcript_alignments.gff3 \
	--segmentSize 100000 --overlapSize 10000 --CPU 20
```

> **species.evm.gff3**

------

### 6 OGS annotation

#### 6.1 OGS annotation updates

> #### ***PASA***
>
> + masked geome (genome.fa)
> + species.evm.gff3
> + stringtie.gtf

```shell
# PASA alignment Assembly
util/gtf_genome_to_cdna_fasta.pl stringtie.gtf genome.fa > transcripts.fasta
bin/seqclean transcripts.fasta

# transcripts alignments
# alignAssembly.config, set up the mysql database name; CPU <= 16
Launch_PASA_pipeline.pl -c alignAssembly.config -C -R -g genome.fa -t transcripts.fasta.clean -T -u transcripts.fasta --ALIGNERS blat --CPU 16

# -------------------------------------------------------------------------------------- #
# two cycles !!! of annotation loading, annotation comparison, and annotation updates
# check gff3
misc_utilities/pasa_gff3_validator.pl species.evm.gff3

# load annotation
scripts/Load_Current_Gene_Annotations.dbi -c alignAssembly.config -g genome.fa -P species.evm.gff3

# update
# annotCompare.config, set up the mysql database name same as alignAssembly.config
Launch_PASA_pipeline.pl -c annotCompare.config -A -g genome.fa -t transcripts.fasta.clean
```

> #### *Collect GFF, cds, PEP*
>
> + gene_structures_post_PASA_updates.gff3

```shell
# rename gff3
# species name, Sfru
python PASA_gff_rename.py gene_structures_post_PASA_updates.gff3 Sfru EVM > Sfru.gff3

# collect
gffread Sfru.gff3 -g genome.fa -x Sfru_cds.fa -y Sfru_pep.fa

# collect no alt gff, cds, pep 
python Collect_no_alt.py pep.fa cds.fa Sfru.gff3
# no_alt.gff3, cds_no_alt.fa, pep_no_alt.fa
```

> + **Sfru.gff3 (Sfru_no_alt.gff3)**
> + **cds.fa (cds_no_alt.fa)**
> + **pep.fa (pep_no_alt.fa)**




#### 6.2 Function annotation

> #### *eggNOG-mapper*
>
> + pep_no_alt.fa

http://eggnog-mapper.embl.de/

------

