# Verticillium_pathogenomics

Documentation of identification of pathogenicity genes in verticillium
Note - all this work was performed in the directory:
/home/groups/harrisonlab/project_files/verticilium_dahliae/pathogenomics

The following is a summary of the work presented in this Readme.

The following processes were applied to Verticillium genomes prior to analysis:
Data qc
Genome assembly
Repeatmasking
Gene prediction
Functional annotation

Analyses performed on these genomes involved BLAST searching for:



## Data extraction

```bash
  cd /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics
  RawDatDir=/home/harrir/projects/pacbio_test/v_dahliae
  OutDir=raw_dna/pacbio/V.dahliae/12008
  mkdir -p $OutDir
  cp -r $RawDatDir/F04_1 $OutDir/.
  cp -r $RawDatDir/G04_1 $OutDir/.
  cp -r $RawDatDir/H04_1 $OutDir/.
  mkdir -p $OutDir/extracted

  cat $OutDir/*/Analysis_Results/*.subreads.fastq > $OutDir/extracted/concatenated_pacbio.fastq
```
```bash
  # For new sequencing run
  RawDat=/home/groups/harrisonlab/raw_data/raw_seq/raw_reads/160404_M004465_0008-ALVUT    
  Species="V.dahliae"
  Strain="12008"
  mkdir -p raw_dna/paired/$Species/$Strain/F
  mkdir -p raw_dna/paired/$Species/$Strain/R
  cp $RawDat/Vd12008_S1_L001_R1_001.fastq.gz raw_dna/paired/$Species/$Strain/F/.
  cp $RawDat/Vd12008_S1_L001_R2_001.fastq.gz raw_dna/paired/$Species/$Strain/R/.
```


## Data qc

programs:
  fastqc
  fastq-mcf
  kmc

Data quality was visualised using fastqc:
```bash
  for RawData in $(ls raw_dna/paired/*/*/*/*.fastq.gz); do
    ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```

Trimming was performed on data to trim adapters from
sequences and remove poor quality data. This was done with fastq-mcf


Trimming was first performed on the strain that had a single run of data:

```bash
  for StrainPath in $(ls -d raw_dna/paired/*/*); do
    ProgDir=/home/fanron/git_repos/tools/seq_tools/rna_qc
    IlluminaAdapters=/home/fanron/git_repos/tools/seq_tools/ncbi_adapters.fa
    ReadsF=$(ls $StrainPath/F/*.fastq*)
    ReadsR=$(ls $StrainPath/R/*.fastq*)
    echo $ReadsF
    echo $ReadsR
    qsub $ProgDir/rna_qc_fastq-mcf.sh $ReadsF $ReadsR $IlluminaAdapters DNA
  done
```


Data quality was visualised once again following trimming:
```bash
  for RawData in $(ls qc_dna/paired/*/*/*/*.fq.gz); do
    ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```

### Identifing read depth

```bash
  for Reads in $(ls raw_dna/pacbio/*/*/extracted/concatenated_pacbio.fastq); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc
    OutDir=$(dirname $Reads)
    qsub $ProgDir/sub_count_nuc.sh 35 $Reads $OutDir
  done
  for Reads in $(ls qc_dna/paired/*/*/*/*_trim.fq.gz); do
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc
    OutDir=$(dirname $Reads)
    qsub $ProgDir/sub_count_nuc.sh 35 $Reads $OutDir
  done
```

The predicted coverage was calculated to be:

```bash
# For PacBio data:
for StrainDir in $(ls -d raw_dna/pacbio/*/* ); do
Strain=$(basename $StrainDir)
printf "$Strain\t"
for File in $(ls qc_dna/paired/*/"$Strain"/*/*.txt); do
echo $(basename $File);
cat $File | tail -n1 | rev | cut -f2 -d ' ' | rev;
done | grep -v '.txt' | awk '{ SUM += $1} END { print SUM }'
done
# For illumina data
for StrainDir in $(ls -d qc_dna/paired/*/* ); do
Strain=$(basename $StrainDir)
printf "$Strain\t"
for File in $(ls qc_dna/paired/*/"$Strain"/*/*.txt); do
echo $(basename $File);
cat $File | tail -n1 | rev | cut -f2 -d ' ' | rev;
done | grep -v '.txt' | awk '{ SUM += $1} END { print SUM }'
done
```

## Assembly

### Canu assembly

```bash
  for Reads in $(ls raw_dna/pacbio/*/*/extracted/concatenated_pacbio.fastq); do
    GenomeSz="35m"
    Strain=$(echo $Reads | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Reads | rev | cut -f4 -d '/' | rev)
    Prefix="$Strain"_canu
    OutDir="assembly/canu/$Organism/$Strain"
    ProgDir=~/git_repos/tools/seq_tools/assemblers/canu
    qsub $ProgDir/submit_canu.sh $Reads $GenomeSz $Prefix $OutDir
  done
```


Assembly stats were collected using quast

```bash
  ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/assembly_qc/quast
  for Assembly in $(ls assembly/canu/*/*/*_canu.contigs.fasta); do
    Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
    OutDir=assembly/canu/$Organism/$Strain/filtered_contigs
    qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  done
```


Polish assemblies using Pilon

```bash
  for Assembly in $(ls assembly/canu/*/*/*_canu.contigs.fasta); do
    Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
    IlluminaDir=$(ls -d qc_dna/paired/$Organism/$Strain)
    TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz);
    TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz);
    OutDir=assembly/canu/$Organism/$Strain/polished
    ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/pilon
    qsub $ProgDir/sub_pilon.sh $Assembly $TrimF1_Read $TrimR1_Read $OutDir
  done
```


After investigation, it was found that contigs didnt need to be split.

Assembly stats were collected using quast

```bash
  ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/assembly_qc/quast
  for Assembly in $(ls assembly/canu/V.dahliae/12008/polished/pilon.fasta); do
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
    OutDir=assembly/canu/$Organism/$Strain/pilon
    qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  done
```
Checking PacBio coverage against Canu assembly

```bash
  Assembly=assembly/canu/V.dahliae/12008/polished/pilon.fasta
  Reads=raw_dna/pacbio/V.dahliae/12008/extracted/concatenated_pacbio.fastq
  OutDir=analysis/genome_alignment/bwa/Verticillium/12008/vs_12008
  ProgDir=/home/fanron/git_repos/tools/seq_tools/genome_alignment/bwa
  qsub $ProgDir/sub_bwa_pacbio.sh $Assembly $Reads $OutDir
```

### Spades Assembly

```bash
  for PacBioDat in $(ls raw_dna/pacbio/*/*/extracted/concatenated_pacbio.fastq); do
    Organism=$(echo $PacBioDat | rev | cut -f4 -d '/' | rev)
    Strain=$(echo $PacBioDat | rev | cut -f3 -d '/' | rev)
    IlluminaDir=$(ls -d qc_dna/paired/$Organism/$Strain)
    TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz);
    TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz);
    OutDir=assembly/spades_pacbio/$Organism/"$Strain"
    echo $TrimR1_Read
    echo $TrimR1_Read
    ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/spades
    qsub $ProgDir/sub_spades_pacbio.sh $PacBioDat $TrimF1_Read $TrimR1_Read $OutDir 20
  done
```

Contigs shorter thaan 500bp were removed from the assembly

```bash
  for Contigs in $(ls assembly/spades_pacbio/*/*/contigs.fasta); do
    AssemblyDir=$(dirname $Contigs)
    mkdir $AssemblyDir/filtered_contigs
    FilterDir=/home/armita/git_repos/tools/seq_tools/assemblers/abyss
    $FilterDir/filter_abyss_contigs.py $Contigs 500 > $AssemblyDir/filtered_contigs/contigs_min_500bp.fasta
  done
```

Checking PacBio coverage against Spades assembly

```bash
  Assembly=assembly/spades_pacbio/V.dahliae/12008/filtered_contigs/contigs_min_500bp.fasta
  Reads=raw_dna/pacbio/V.dahliae/12008/extracted/concatenated_pacbio.fastq
  OutDir=analysis/genome_alignment/bwa/Verticillium/12008/vs_spades_assembly
  ProgDir=/home/fanron/git_repos/tools/seq_tools/genome_alignment/bwa
  qsub $ProgDir/sub_bwa_pacbio.sh $Assembly $Reads $OutDir
```

## Merging pacbio and hybrid assemblies

```bash
  for PacBioAssembly in $(ls assembly/canu/*/*/polished/pilon.fasta); do
    Organism=$(echo $PacBioAssembly | rev | cut -f4 -d '/' | rev)
    Strain=$(echo $PacBioAssembly | rev | cut -f3 -d '/' | rev)
    HybridAssembly=$(ls assembly/spades_pacbio/$Organism/$Strain/contigs.fasta)
    AnchorLength=500000
    OutDir=assembly/merged_canu_spades/$Organism/"$Strain"
    ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/quickmerge
    qsub $ProgDir/sub_quickmerge.sh $PacBioAssembly $HybridAssembly $OutDir $AnchorLength
  done
```

This merged assembly was polished using Pilon

```bash
  for Assembly in $(ls assembly/merged_canu_spades/*/*/merged.fasta); do
    Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
    IlluminaDir=$(ls -d qc_dna/paired/$Organism/$Strain)
    TrimF1_Read=$(ls $IlluminaDir/F/*_trim.fq.gz);
    TrimR1_Read=$(ls $IlluminaDir/R/*_trim.fq.gz);
    OutDir=assembly/merged_canu_spades/$Organism/$Strain/polished
    ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/pilon
    qsub $ProgDir/sub_pilon.sh $Assembly $TrimF1_Read $TrimR1_Read $OutDir
  done
```

Contigs were renamed in accordance with ncbi recomendations.

```bash
  ProgDir=~/git_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
  touch tmp.csv
  for Assembly in $(ls assembly/merged_canu_spades/*/*/polished/pilon.fasta); do
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    OutDir=assembly/merged_canu_spades/$Organism/$Strain/filtered_contigs
    mkdir -p $OutDir
    $ProgDir/remove_contaminants.py --inp $Assembly --out $OutDir/"$Strain"_contigs_renamed.fasta --coord_file tmp.csv
  done
  rm tmp.csv
```

Assembly stats were collected using quast

```bash
  ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/assembly_qc/quast
  for Assembly in $(ls assembly/merged_canu_spades/*/*/filtered_contigs/*_contigs_renamed.fasta); do
    Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
    OutDir=$(dirname $Assembly)
    qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  done
```

Assembly                     12008_contigs_renamed

contigs (>= 0 bp)          104                             
contigs (>= 1000 bp)       104                                        
Total length (>= 0 bp)       35100962                              
Total length (>= 1000 bp)    35100962                                   
contigs                     104                                         
Largest contig               2438101                                   
Total length                 35100962                                
GC (%)                       54.53                  
N50                          746680                                    
N75                          389743                                     
L50                          16                                               
L75                          32                                               
N's per 100 kbp             0.00                   

A Bioproject and Biosample was made with NCBI genbank for submission of genomes.
Following the creation of these submissions, the .fasta assembly was uploaded
through the submission portal. A note was provided requesting that the assembly
be run through the contamination screen to aid a more detailed resubmission in
future. The returned FCSreport.txt was downloaded from the NCBI webportal and
used to correct the assembly to NCBI standards.

NCBI reports (FCSreport.txt) were manually downloaded to the following loactions:

```bash
  for Assembly in $(ls assembly/merged_canu_spades/V.dahliae/12008/filtered_contigs/12008_contigs_renamed.fasta); do
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
    NCBI_report_dir=genome_submission/$Organism/$Strain/initial_submission
    mkdir -p $NCBI_report_dir
  done
```
These downloaded files were used to correct assemblies:

```bash
for Assembly in $(ls assembly/merged_canu_spades/V.dahliae/12008/filtered_contigs/12008_contigs_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)
echo "$Organism - $Strain"
NCBI_report=$(ls genome_submission/$Organism/$Strain/initial_submission/FCSreport.txt)
OutDir=assembly/merged_canu_spades/$Organism/$Strain/ncbi_edits
mkdir -p $OutDir
ProgDir=/home/fanron/git_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
$ProgDir/remove_contaminants.py --inp $Assembly --out $OutDir/12008_contigs_renamed.fasta --coord_file $NCBI_report > $OutDir/log.txt
done
```
Quast was used to collect details on these assemblies again

```bash
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
for Assembly in $(ls assembly/merged_canu_spades/*/*/ncbi_edits/12008_contigs_renamed.fasta); do
Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
echo "$Organism - $Strain"
OutDir=assembly/merged_canu_spades/$Organism/$Strain/ncbi_edits
qsub $ProgDir/sub_quast.sh $Assembly $OutDir
done
```
All statistics are based on contigs of size >= 500 bp, unless otherwise noted (e.g., "# contigs (>= 0 bp)" and "Total length (>= 0 bp)" include all contigs).

Assembly                   12008_contigs_renamed  12008_contigs_renamed broken
# contigs (>= 0 bp)        103                    103
# contigs (>= 1000 bp)     103                    103
Total length (>= 0 bp)     35057408               35057408
Total length (>= 1000 bp)  35057408               35057408
# contigs                  103                    103
Largest contig             2438101                2438101
Total length               35057408               35057408
GC (%)                     54.57                  54.57
N50                        746680                 746680
N75                        389743                 389743
L50                        16                     16
L75                        32                     32
# N's per 100 kbp          0.00                   0.00



Checking PacBio coverage against merged assembly

```bash
  Assembly=assembly/merged_canu_spades/V.dahliae/12008/ncbi_edits/12008_contigs_renamed.fasta
  Reads=raw_dna/pacbio/V.dahliae/12008/extracted/concatenated_pacbio.fastq
  OutDir=analysis/genome_alignment/bwa/Verticillium/12008/ncbi_12008
  ProgDir=/home/fanron/git_repos/tools/seq_tools/genome_alignment/bwa
  qsub $ProgDir/sub_bwa_pacbio.sh $Assembly $Reads $OutDir
```


##Repeatmasking

Repeat masking was performed and used the following programs:
  Repeatmasker
  Repeatmodeler

The best assemblies were used to perform repeatmasking

```bash
  ProgDir=/home/fanron/git_repos/tools/seq_tools/repeat_masking
  for BestAss in $(ls assembly/merged_canu_spades/*/*/ncbi_edits/12008_contigs_renamed.fasta); do
    Organism=$(echo $BestAss | rev | cut -d "/" -f4 | rev)
    Strain=$(echo $BestAss | rev | cut -d "/" -f3 | rev)
    OutDir=repeat_masked/$Organism/$Strain/ncbi_filtered_contigs_repmask
    qsub $ProgDir/rep_modeling.sh $BestAss $OutDir
    qsub $ProgDir/transposonPSI.sh $BestAss $OutDir
  done
```

The number of bases masked by transposonPSI and Repeatmasker were summarised
using the following commands:

```bash
  for RepDir in $(ls -d repeat_masked/V.*/*/ncbi*); do
    Strain=$(echo $RepDir | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $RepDir | rev | cut -f3 -d '/' | rev)  
    RepMaskGff=$(ls $RepDir/*_contigs_hardmasked.gff)
    TransPSIGff=$(ls $RepDir/*_contigs_unmasked.fa.TPSI.allHits.chains.gff3)
    printf "$Organism\t$Strain\n"
    printf "The number of bases masked by RepeatMasker:\t"
    sortBed -i $RepMaskGff | bedtools merge | awk -F'\t' 'BEGIN{SUM=0}{ SUM+=$3-$2 }END{print SUM}'
    printf "The number of bases masked by TransposonPSI:\t"
    sortBed -i $TransPSIGff | bedtools merge | awk -F'\t' 'BEGIN{SUM=0}{ SUM+=$3-$2 }END{print SUM}'
    printf "The total number of masked bases are:\t"
    cat $RepMaskGff $TransPSIGff | sortBed | bedtools merge | awk -F'\t' 'BEGIN{SUM=0}{ SUM+=$3-$2 }END{print SUM}'
    echo
  done
```
Results were as follows:
```
V.dahliae	12008
The number of bases masked by RepeatMasker:	3280336
The number of bases masked by TransposonPSI:	859780
The total number of masked bases are:	3372268

V.dahliae	51
The number of bases masked by RepeatMasker:	1195031
The number of bases masked by TransposonPSI:	310921
The total number of masked bases are:	1377060

V.dahliae	53
The number of bases masked by RepeatMasker:	689605
The number of bases masked by TransposonPSI:	221954
The total number of masked bases are:	863683

V.dahliae	58
The number of bases masked by RepeatMasker:	1258760
The number of bases masked by TransposonPSI:	360475
The total number of masked bases are:	1418679

V.dahliae	61
The number of bases masked by RepeatMasker:	1574548
The number of bases masked by TransposonPSI:	407135
The total number of masked bases are:	1726438
```

```bash
  for File in $(ls repeat_masked/*/*/ncbi_filtered_contigs_repmask/*_contigs_softmasked.fa); do
    OutDir=$(dirname $File)
    TPSI=$(ls $OutDir/*_contigs_unmasked.fa.TPSI.allHits.chains.gff3)
    OutFile=$(echo $File | sed 's/_contigs_softmasked.fa/_contigs_softmasked_repeatmasker_TPSI_appended.fa/g')
    bedtools maskfasta -soft -fi $File -bed $TPSI -fo $OutFile
    echo "$OutFile"
    echo "Number of masked bases:"
    cat $OutFile | grep -v '>' | tr -d '\n' | awk '{print $0, gsub("[a-z]", ".")}' | cut -f2 -d ' '
  done
```


## Gene Prediction

  Pre-gene prediction
    - Quality of genome assemblies were assessed using Cegma to see how many core eukaryotic genes can be identified.
  Gene model training
    - Gene models were trained using assembled RNAseq data as part of the Braker1 pipeline
  Gene prediction
    - Gene models were used to predict genes in genomes as part of the the Braker1 pipeline. This used RNAseq data as hints for gene models.


### Pre-gene prediction

Quality of genome assemblies was assessed by looking for the gene space in the assemblies.

```bash
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/cegma
  cd /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics
  for Genome in $(ls repeat_masked/V.*/*/ncbi*/*_contigs_softmasked.fa); do
    echo $Genome;
    OutDir=gene_pre/ncbi_cegma
    mkdir -p gene_pre/ncbi_cegma
    qsub $ProgDir/sub_cegma.sh $Genome dna $OutDIr;
  done
```
*** Number of cegma genes present and complete: 95.56%
** Number of cegma genes present and partial: 98.39%

```bash
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/cegma
  cd /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics
  for Genome in $(ls repeat_masked/V.*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    echo $Genome;
    OutDir=gene_pre/ncbi_cegma
    mkdir -p gene_pre/ncbi_cegma
    qsub $ProgDir/sub_cegma.sh $Genome dna;
  done
```
*** Number of cegma genes present and complete: 95.56%
** Number of cegma genes present and partial: 98.39%

Outputs were summarised using the commands:
```bash
  for File in $(ls gene_pre/ncbi_cegma/V.*/12008/*_dna_cegma.completeness_report); do
    Strain=$(echo $File | rev | cut -f2 -d '/' | rev);
    Species=$(echo $File | rev | cut -f3 -d '/' | rev);
    printf "$Species\t$Strain\n";
    cat $File | head -n18 | tail -n+4;printf "\n";
  done > gene_pred/cegma/cegma_results_dna_summary.txt

less gene_pred/cegma/cegma_results_dna_summary.txt

#    These results are based on the set of genes selected by Genis Parra   #
#    Key:                                                                  #
#    Prots = number of 248 ultra-conserved CEGs present in genome          #
#    %Completeness = percentage of 248 ultra-conserved CEGs present        #
#    Total = total number of CEGs present including putative orthologs     #
#    Average = average number of orthologs per CEG                         #
#    %Ortho = percentage of detected CEGS that have more than 1 ortholog   #
```
There are 237 complete and 7 partial core eukaryotic genes (out of total 248 genes) present in my assembly


Busco has replaced CEGMA and was run to check gene space in assemblies

```bash
for Assembly in $(ls repeat_masked/V.*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/sordariomyceta_odb9)
OutDir=../tmp/gene_pred/busco/$Organism/$Strain/assembly
qsub $ProgDir/sub_busco2.sh $Assembly $BuscoDB $OutDir
done
```

```bash
for File in $(ls ../tmp/gene_pred/busco/*/*/assembly/*/short_summary_*.txt); do
Strain=$(echo $File| rev | cut -d '/' -f4 | rev)
Organism=$(echo $File | rev | cut -d '/' -f5 | rev)
Complete=$(cat $File | grep "(C)" | cut -f2)
Fragmented=$(cat $File | grep "(F)" | cut -f2)
Duplicated=$(cat $File | grep "(D)" | cut -f2)
Missing=$(cat $File | grep "(M)" | cut -f2)
Total=$(cat $File | grep "Total" | cut -f2)
echo -e "$Organism\t$Strain\t$Complete\t$Fragmented\t$Duplicated\t$Missing\t$Total"
done
```




## Gene prediction


  make folders for RNA_seq data

  mkdir -p raw_rna/paired/V.dahiae/12008PDA/F
  mkdir -p raw_rna/paired/V.dahiae/12008PDA/R
  mkdir -p raw_rna/paired/V.dahiae/12008CD/F
  mkdir -p raw_rna/paired/V.dahiae/12008CD/R

Copy raw RNA_seq data from miseq_data folder to pathogenomic folder

This contained the following data:

  ```
  12008PDA

  12008-PDA_S2_L001_R1_001.fastq.gz  12008-PDA_S2_L001_R2_001.fastq.gz
  12008-PDA_S2_L001_R1_001.fastq  12008-PDA_S2_L001_R2_001.fastq

  12008CD

  12008-CD_S1_L001_R1_001.fastq.gz   12008-CD_S1_L001_R2_001.fastq.gz
  12008-CD_S1_L001_R1_001.fastq   12008-CD_S1_L001_R2_001.fastq
```
Perform qc of RNAseq data

```bash
  for FilePath in $(ls -d raw_rna/paired/V.*/12008PDA); do
    echo $FilePath;
    FileF=$(ls $FilePath/F/*.gz);
    FileR=$(ls $FilePath/R/*.gz);
    IlluminaAdapters=/home/fanron/git_repos/tools/seq_tools/ncbi_adapters.fa; ProgDir=/home/fanron/git_repos/tools/seq_tools/rna_qc;
    qsub $ProgDir/rna_qc_fastq-mcf.sh $FileF $FileR $IlluminaAdapters RNA;
  done
```

```bash
  for FilePath in $(ls -d raw_rna/paired/V.*/12008CD); do
    echo $FilePath;
    FileF=$(ls $FilePath/F/*.gz);
    FileR=$(ls $FilePath/R/*.gz);
    IlluminaAdapters=/home/fanron/git_repos/tools/seq_tools/ncbi_adapters.fa; ProgDir=/home/fanron/git_repos/tools/seq_tools/rna_qc;
    qsub $ProgDir/rna_qc_fastq-mcf.sh $FileF $FileR $IlluminaAdapters RNA;
  done
```
Data quality was visualised using fastqc:

```bash
  for RawData in $(ls qc_rna/paired/V.*/12008PDA/R/*.fq.gz); do
  ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```
```bash
  for RawData in $(ls qc_rna/paired/V.*/12008PDA/F/*.fq.gz); do
  ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```
```bash
  for RawData in $(ls qc_rna/paired/V.*/12008CD/R/*.fq.gz); do
  ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```
```bash
  for RawData in $(ls qc_rna/paired/V.*/12008CD/F/*.fq.gz); do
  ProgDir=/home/fanron/git_repos/tools/seq_tools/dna_qc
    echo $RawData;
    qsub $ProgDir/run_fastqc.sh $RawData
  done
```
#### Aligning

Insert sizes of the RNA seq library were unknown until a draft alignment could
be made. To do this tophat and cufflinks were run, aligning the reads against a
single genome. The fragment length and stdev were printed to stdout while
cufflinks was running.

```bash
  for Assembly in $(ls repeat_masked/*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    for RNADir in $(ls -d qc_rna/paired/V.*/12008PDA); do
      Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
      echo "$Timepoint"
      FileF=$(ls $RNADir/F/*_trim.fq.gz)
      FileR=$(ls $RNADir/R/*_trim.fq.gz)
      OutDir=ncbi_alignment/$Organism/$Strain/$Timepoint
      ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
      qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir
    done
  done
```
81.5% overall read mapping rate.
72.8% concordant pair alignment rate.



```bash
  for Assembly in $(ls repeat_masked/*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    for RNADir in $(ls -d qc_rna/paired/V.*/12008CD); do
      Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
      echo "$Timepoint"
      FileF=$(ls $RNADir/F/*_trim.fq.gz)
      FileR=$(ls $RNADir/R/*_trim.fq.gz)
      OutDir=ncbi_alignment/$Organism/$Strain/$Timepoint
      ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
      qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir
    done
  done
```
80.4% overall read mapping rate.
70.9% concordant pair alignment rate.



Alignments were concatenated prior to running cufflinks:
Cufflinks was run to produce the fragment length and stdev statistics:


if run the commands in a node other than cluster, using the script:

```bash
qlogin -pe smp 8 -l virtual_free=1G
```
```bash
for Assembly in $(ls /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics/repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '12008'); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
for AcceptedHits in $(ls /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics/alignment/$Organism/$Strain/*/accepted_hits.bam); do
Timepoint=$(echo $AcceptedHits | rev | cut -f2 -d '/' | rev)
echo $Timepoint
OutDir=/home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics/gene_pred/cufflinks/$Organism/$Strain/"$Timepoint"_prelim
ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
# qsub $ProgDir/sub_cufflinks.sh $AcceptedHits $OutDir
cufflinks -o $OutDir/cufflinks -p 8 --max-intron-length 4000 $AcceptedHits 2>&1 | tee $OutDir/cufflinks/cufflinks.log
done
done
```


if run the commands on cluster other than a node:

```bash
# qlogin -pe smp 8 -l virtual_free=1G
```
```bash
  for Assembly in $(ls repeat_masked/*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  for AcceptedHits in $(ls ncbi_alignment/$Organism/$Strain/*/accepted_hits.bam); do
  Timepoint=$(echo $AcceptedHits | rev | cut -f2 -d '/' | rev)
  echo $Timepoint
  OutDir=gene_pred/ncbi_cufflinks/$Organism/$Strain/"$Timepoint"_prelim
  ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
  qsub $ProgDir/sub_cufflinks.sh $AcceptedHits $OutDir
  # cufflinks -o $OutDir/cufflinks -p 8 --max-intron-length 4000 $AcceptedHits 2>&1 | tee $OutDir/cufflinks/cufflinks.log
  done
  done
  ```
    12008PDA
    > Processed 18229 loci.                        [*************************] 100%
    > Map Properties:
    >       Normalized Map Mass: 11153744.78
    >       Raw Map Mass: 11153744.78
    >       Fragment Length Distribution: Empirical (learned)
    >                     Estimated Mean: 233.34
    >                  Estimated Std Dev: 59.60
    [17:30:39] Assembling transcripts and estimating abundances.
    > Processed 18364 loci.                        [*************************] 100%
    The Estimated Mean: 233.34 allowed calculation of of the mean insert gap to be
    -247bp 233-(240*2) where 240? was the mean read length. This was provided to tophat
    on a second run (as the -r option) along with the fragment length stdev to
    increase the accuracy of mapping.


    12008CD
    > Processed 17114 loci.                        [*************************] 100%
    > Map Properties:
    >       Normalized Map Mass: 8310303.37
    >       Raw Map Mass: 8310303.37
    >       Fragment Length Distribution: Empirical (learned)
    >                     Estimated Mean: 224.61
    >                  Estimated Std Dev: 62.76
    [14:44:25] Assembling transcripts and estimating abundances.
    > Processed 17260 loci.                        [*************************] 100%

    The Estimated Mean: 224.61 allowed calculation of of the mean insert gap to be
    -255bp 225-(240*2) where 240? was the mean read length. This was provided to tophat
    on a second run (as the -r option) along with the fragment length stdev to
    increase the accuracy of mapping.

Insert sizes are probabely larger than fragement sizes

Then Rnaseq data was aligned to each genome assembly:

```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  for RNADir in $(ls -d qc_rna/paired/*/12008PDA | grep -v -e '_rep'); do
  Timepoint_PDA=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
  echo "$Timepoint_PDA"
  FileF=$(ls $RNADir/F/*_trim.fq.gz)
  FileR=$(ls $RNADir/R/*_trim.fq.gz)
  OutDir=alignment/$Organism/$Strain/$Timepoint_PDA
  InsertGap='-247'
  InsertStdDev='60'
  Jobs=$(qstat | grep 'tophat' | grep 'qw' | wc -l)
  while [ $Jobs -gt 1 ]; do
  sleep 10
  printf "."
  Jobs=$(qstat | grep 'tophat' | grep 'qw' | wc -l)
  done
  printf "\n"
  ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
  qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir $InsertGap $InsertStdDev
  done
  done

  cd alignment/repeat_masked/
  mkdir 12008PDA_accurate
  cp -r 12008PDA/* 12008PDA_accurate/
```
  81.5% overall read mapping rate.
  72.8% concordant pair alignment rate.


```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  for RNADir in $(ls -d qc_rna/paired/*/12008CD | grep -v -e '_rep'); do
  Timepoint_CD=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
  echo "$Timepoint_CD"
  FileF=$(ls $RNADir/F/*_trim.fq.gz)
  FileR=$(ls $RNADir/R/*_trim.fq.gz)
  OutDir=alignment/$Organism/$Strain/$Timepoint_CD
  InsertGap='-255'
  InsertStdDev='63'
  Jobs=$(qstat | grep 'tophat' | grep 'qw' | wc -l)
  while [ $Jobs -gt 1 ]; do
  sleep 10
  printf "."
  Jobs=$(qstat | grep 'tophat' | grep 'qw' | wc -l)
  done
  printf "\n"
  ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
  qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir $InsertGap $InsertStdDev
  done
  done

  cd alignment/repeat_masked/
  mkdir 12008CD_accurate
  cp -r 12008CD/* 12008CD_accurate/
```

  80.4% overall read mapping rate.
  70.8% concordant pair alignment rate.

#### Braker prediction

Before braker predictiction was performed, I double checked that I had the genemark key in my user area and copied it over from the genemark install directory:

```bash
ls ~/.gm_key
cp /home/armita/prog/genemark/gm_key_64 ~/.gm_key
```

Braker predictiction was performed using softmasked genome, not unmasked one.

```bash
  for Assembly in $(ls repeat_masked/*/*/ncbi*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa | grep '12008'); do
  Jobs=$(qstat | grep 'tophat' | grep -w 'r' | wc -l)
  while [ $Jobs -gt 1 ]; do
  sleep 10
  printf "."
  Jobs=$(qstat | grep 'tophat' | grep -w 'r' | wc -l)
  done
  printf "\n"
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  OutDir=gene_pred/braker/$Organism/"$Strain"_braker_sixth
  AcceptedHits=alignment/$Organism/$Strain/concatenated/concatenated.bam
  mkdir -p ncbi_alignment/$Organism/$Strain/concatenated
  # samtools merge -f $AcceptedHits \
  #   alignment/$Organism/$Strain/*CD/accepted_hits.bam \
  #   alignment/$Organism/$Strain/*PDA/accepted_hits.bam
  GeneModelName="$Organism"_"$Strain"_braker_sixth
  rm -r /home/armita/prog/augustus-3.1/config/species/$GeneModelName
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/braker1
  qsub $ProgDir/sub_braker_fungi.sh $Assembly $OutDir $AcceptedHits $GeneModelName
  done
```

Amino acid sequences and gff files were extracted from Braker1 output.

```bash
  for File in $(ls gene_pred/braker/V.dahliae/12008_publication/*/augustus.gff); do
  getAnnoFasta.pl $File
  OutDir=$(dirname $File)
  echo "##gff-version 3" > $OutDir/augustus_extracted.gff
  cat $File | grep -v '#' >> $OutDir/augustus_extracted.gff
  done
```

## Supplimenting Braker gene models with CodingQuary genes

Additional genes were added to Braker gene predictions, using CodingQuary in
pathogen mode to predict additional regions.

Fistly, aligned RNAseq data was assembled into transcripts using Cufflinks.

Note - cufflinks doesn't always predict direction of a transcript and
therefore features can not be restricted by strand when they are intersected.


```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  OutDir=gene_pred/cufflinks/$Organism/$Strain/concatenated_prelim
  mkdir -p $OutDir
  AcceptedHits=alignment/$Organism/$Strain/concatenated/concatenated.bam
  ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
  qsub $ProgDir/sub3_cufflinks.sh $AcceptedHits $OutDir
  done
```

Secondly, genes were predicted using CodingQuary:

```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  OutDir=gene_pred/codingquary1/$Organism/$Strain
  CufflinksGTF=gene_pred/cufflinks/$Organism/$Strain/concatenated_prelim/transcripts.gtf
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/codingquary
  qsub $ProgDir/sub1_CodingQuary.sh $Assembly $CufflinksGTF $OutDir
  done
```

Then, additional transcripts were added to Braker gene models, when CodingQuary
genes were predicted in regions of the genome, not containing Braker gene
models:
<!--
```bash
  # for BrakerGff in $(ls gene_pred/braker/F.*/*_braker_new/*/augustus.gff3 | grep -w -e 'Fus2'); do
  for BrakerGff in $(ls gene_pred/braker/V.*/12008_braker_sixth/*/augustus.gff3); do
  Strain=$(echo $BrakerGff| rev | cut -d '/' -f3 | rev | sed 's/_braker_sixth//g')
  Organism=$(echo $BrakerGff | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  Assembly=$(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa)
  CodingQuaryGff=gene_pred/codingquary1/$Organism/$Strain/out/PredictedPass.gff3
  PGNGff=gene_pred/codingquary1/$Organism/$Strain/out/PGN_predictedPass.gff3
  AddDir=gene_pred/codingquary1/$Organism/$Strain/additional
  FinalDir=gene_pred/codingquary1/$Organism/$Strain/final
  AddGenesList=$AddDir/additional_genes.txt
  AddGenesGff=$AddDir/additional_genes.gff
  FinalGff=$AddDir/combined_genes.gff
  mkdir -p $AddDir
  mkdir -p $FinalDir

  bedtools intersect -v -a $CodingQuaryGff -b $BrakerGff | grep 'gene'| cut -f2 -d'=' | cut -f1 -d';' > $AddGenesList
  bedtools intersect -v -a $PGNGff -b $BrakerGff | grep 'gene'| cut -f2 -d'=' | cut -f1 -d';' >> $AddGenesList
  ProgDir=/home/gomeza/git_repos/emr_repos/tools/seq_tools/feature_annotation
  $ProgDir/gene_list_to_gff.pl $AddGenesList $CodingQuaryGff CodingQuarry_v2.0 ID CodingQuary > $AddGenesGff
  $ProgDir/gene_list_to_gff.pl $AddGenesList $PGNGff PGNCodingQuarry_v2.0 ID CodingQuary >> $AddGenesGff
  ProgDir=/home/gomeza/git_repos/emr_repos/tools/gene_prediction/codingquary

  $ProgDir/add_CodingQuary_features.pl $AddGenesGff $Assembly > $FinalDir/final_genes_CodingQuary.gff3
  $ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_CodingQuary.gff3 $FinalDir/final_genes_CodingQuary
  cp $BrakerGff $FinalDir/final_genes_Braker.gff3
  $ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_Braker.gff3 $FinalDir/final_genes_Braker
  cat $FinalDir/final_genes_Braker.pep.fasta $FinalDir/final_genes_CodingQuary.pep.fasta | sed -r 's/\*/X/g' > $FinalDir/final_genes_combined.pep.fasta
  cat $FinalDir/final_genes_Braker.cdna.fasta $FinalDir/final_genes_CodingQuary.cdna.fasta > $FinalDir/final_genes_combined.cdna.fasta
  cat $FinalDir/final_genes_Braker.gene.fasta $FinalDir/final_genes_CodingQuary.gene.fasta > $FinalDir/final_genes_combined.gene.fasta
  cat $FinalDir/final_genes_Braker.upstream3000.fasta $FinalDir/final_genes_CodingQuary.upstream3000.fasta > $FinalDir/final_genes_combined.upstream3000.fasta

  GffBraker=$FinalDir/final_genes_CodingQuary.gff3
  GffQuary=$FinalDir/final_genes_Braker.gff3
  GffAppended=$FinalDir/final_genes_appended.gff3
  cat $GffBraker $GffQuary > $GffAppended

  # cat $BrakerGff $AddDir/additional_gene_parsed.gff3 | bedtools sort > $FinalGff
  done
```

The final number of genes per isolate was observed using:

```bash
  # for DirPath in $(ls -d gene_pred/codingquary/V.*/*/final_PDA); do
  #echo $DirPath;
  #cat $DirPath/final_genes_Braker.pep.fasta | grep '>' | wc -l;
  #cat $DirPath/final_genes_CodingQuary.pep.fasta | grep '>' | wc -l;
  #cat $DirPath/final_genes_combined.pep.fasta | grep '>' | wc -l;
  #echo "";
  #done
```
  #gene_pred/codingquary/V.dahliae/12008/final_CD
  #9871
  #704
  #10575
  #gene_pred/codingquary/V.dahliae/12008/final_PDA
  #9871
  #585
  #10456


```bash
  for DirPath in $(ls -d gene_pred/*/V.*/*/final); do
  Strain=$(echo $DirPath| rev | cut -d '/' -f2 | rev)
  Organism=$(echo $DirPath | rev | cut -d '/' -f3 | rev)
  Braker=$(cat $DirPath/final_genes_Braker.pep.fasta | grep '>' | wc -l);
  CodingQuary=$(cat $DirPath/final_genes_CodingQuary.pep.fasta | grep '>' | wc -l);
  TotalProteins=$(cat $DirPath/final_genes_combined.pep.fasta | grep '>' | wc -l);
  TotalGenes=$(cat $DirPath/final_genes_combined.pep.fasta | grep '>' | cut -f1 -d '.' | sort | uniq | wc -l);
  printf "$Organism\t$Strain\t$Braker\t$CodingQuary\t$TotalProteins\t$TotalGenes\n";
  done
```

```
V.dahliae	12008	9881	720	10601	10440
V.dahliae	51	9655	732	10387	10296
V.dahliae	53	9718	716	10434	10332
V.dahliae	58	9458	545	10003	9925
V.dahliae	61	9457	550	10007	9936
``` -->


Then, additional transcripts were added to Braker gene models, when CodingQuary
genes were predicted in regions of the genome, not containing Braker gene
models:

Note - Ensure that the "TPSI_appended.fa" assembly file is correct.

```bash
for BrakerGff in $(ls gene_pred/braker/*/*/*_publication/augustus.gff3 | grep '12008'); do
Strain=$(echo $BrakerGff| rev | cut -d '/' -f3 | rev | sed 's/_publication//g')
Organism=$(echo $BrakerGff | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
Assembly=$(ls repeat_masked/$Organism/$Strain/ncbi_filtered_contigs_repmask/*_softmasked_repeatmasker_TPSI_appended.fa)
CodingQuaryGff=gene_pred/codingquary1/$Organism/$Strain/out/PredictedPass.gff3
PGNGff=gene_pred/codingquary1/$Organism/$Strain/out/PGN_predictedPass.gff3
AddDir=gene_pred/codingquary1/$Organism/$Strain/additional
FinalDir=gene_pred/final/$Organism/$Strain/final
AddGenesList=$AddDir/additional_genes.txt
AddGenesGff=$AddDir/additional_genes.gff
FinalGff=$AddDir/combined_genes.gff
mkdir -p $AddDir
mkdir -p $FinalDir

bedtools intersect -v -a $CodingQuaryGff -b $BrakerGff | grep 'gene'| cut -f2 -d'=' | cut -f1 -d';' > $AddGenesList
bedtools intersect -v -a $PGNGff -b $BrakerGff | grep 'gene'| cut -f2 -d'=' | cut -f1 -d';' >> $AddGenesList
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation
$ProgDir/gene_list_to_gff.pl $AddGenesList $CodingQuaryGff CodingQuarry_v2.0 ID CodingQuary > $AddGenesGff
$ProgDir/gene_list_to_gff.pl $AddGenesList $PGNGff PGNCodingQuarry_v2.0 ID CodingQuary >> $AddGenesGff
ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
# -
# This section is edited
$ProgDir/add_CodingQuary_features.pl $AddGenesGff $Assembly > $AddDir/add_genes_CodingQuary_unspliced.gff3
$ProgDir/correct_CodingQuary_splicing.py --inp_gff $AddDir/add_genes_CodingQuary_unspliced.gff3 > $FinalDir/final_genes_CodingQuary.gff3
# -
$ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_CodingQuary.gff3 $FinalDir/final_genes_CodingQuary
cp $BrakerGff $FinalDir/final_genes_Braker.gff3
$ProgDir/gff2fasta.pl $Assembly $FinalDir/final_genes_Braker.gff3 $FinalDir/final_genes_Braker
cat $FinalDir/final_genes_Braker.pep.fasta $FinalDir/final_genes_CodingQuary.pep.fasta | sed -r 's/\*/X/g' > $FinalDir/final_genes_combined.pep.fasta
cat $FinalDir/final_genes_Braker.cdna.fasta $FinalDir/final_genes_CodingQuary.cdna.fasta > $FinalDir/final_genes_combined.cdna.fasta
cat $FinalDir/final_genes_Braker.gene.fasta $FinalDir/final_genes_CodingQuary.gene.fasta > $FinalDir/final_genes_combined.gene.fasta
cat $FinalDir/final_genes_Braker.upstream3000.fasta $FinalDir/final_genes_CodingQuary.upstream3000.fasta > $FinalDir/final_genes_combined.upstream3000.fasta


GffBraker=$FinalDir/final_genes_Braker.gff3
GffQuary=$FinalDir/final_genes_CodingQuary.gff3
GffAppended=$FinalDir/final_genes_appended.gff3
cat $GffBraker $GffQuary > $GffAppended
done
```

In preperation for submission to ncbi, gene models were renamed and duplicate gene features were identified and removed.
 * no duplicate genes were identified


```bash
  for GffAppended in $(ls gene_pred/final/*/*/final/final_genes_appended.gff3); do
    Strain=$(echo $GffAppended | rev | cut -d '/' -f3 | rev)
    Organism=$(echo $GffAppended | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    FinalDir=gene_pred/final/$Organism/$Strain/final
    GffFiltered=$FinalDir/filtered_duplicates.gff
    ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
    $ProgDir/remove_dup_features.py --inp_gff $GffAppended --out_gff $GffFiltered
    GffRenamed=$FinalDir/final_genes_appended_renamed.gff3
    LogFile=$FinalDir/final_genes_appended_renamed.log
    ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/codingquary
    $ProgDir/gff_rename_genes.py --inp_gff $GffFiltered --conversion_log $LogFile > $GffRenamed
    rm $GffFiltered
    Assembly=$(ls repeat_masked/$Organism/$Strain/ncbi_filtered_contigs_repmask/*_softmasked_repeatmasker_TPSI_appended.fa)
    $ProgDir/gff2fasta.pl $Assembly $GffRenamed gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed
    # The proteins fasta file contains * instead of Xs for stop codons, these should
    # be changed
    sed -i 's/\*/X/g' gene_pred/final/$Organism/$Strain/final/final_genes_appended_renamed.pep.fasta
  done
```



<!--
## ORF finder

The genome was searched in six reading frames for any start codon and following
translated identification of a start codon translating sequence until a stop
codon was found. This is based upon the atg.pl script used in paper describing
the P. infestans genome. Additional functionality was added to this script by
also printing ORFs in .gff format.


```bash
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
  for Genome in $(ls repeat_masked/*/*/ncbi*/*_contigs_unmasked.fa | grep -w -e '12008'); do
    qsub $ProgDir/run_ORF_finder.sh $Genome
    OutDir=gene_pre/ORF_Finder
    mkdir $OutDir
  done
```
The Gff files from the the ORF finder are not in true Gff3 format. These were
corrected using the following commands:

```bash
  ProgDir=~/git_repos/tools/seq_tools/feature_annotation
  for ORF_Gff in $(ls gene_pred/ORF_finder/*/*/*_ORF.gff | grep -v '_F_atg_' | grep -v '_R_atg_'); do
    ORF_Gff_mod=$(echo $ORF_Gff | sed 's/_ORF.gff/_ORF_corrected.gff3/g')
    echo ""
    echo "Correcting the following file:"
    echo $ORF_Gff
    echo "Redirecting to:"
    echo $ORF_Gff_mod
    $ProgDir/gff_corrector.pl $ORF_Gff > $ORF_Gff_mod
  done
```

The final number of genes per isolate was observed using:
```bash
for DirPath in $(ls -d gene_pred/ORF_finder/*/$Strain); do
echo $DirPath
cat $DirPath/*aa_cat.fa | grep '>' | wc -l
echo ""
done
```
gene_pred/ORF_finder/V.dahliae/12008
ORF_finder
329493
-->

#Functional annotation

## A) Interproscan

Interproscan was used to give gene models functional annotations.
Annotation was run using the commands below:

Note: This is a long-running script. As such, these commands were run using
'screen' to allow jobs to be submitted and monitored in the background.
This allows the session to be disconnected and reconnected over time.

Screen ouput detailing the progress of submission of interporscan jobs
was redirected to a temporary output file named interproscan_submission.log .

```bash
  screen -a
  cd /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/interproscan
  # for Genes in $(ls gene_pred/codingquary1/V.*/*/*/final_genes_combined.pep.fasta); do
  for Genes in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
  echo $Genes
  $ProgDir/sub_interproscan.sh $Genes
  done 2>&1 | tee -a interproscan_submisison.log
```

Following interproscan annotation split files were combined using the following
commands:

```bash
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/interproscan
  for Proteins in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
    Strain=$(echo $Proteins | rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Proteins | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    echo $Strain
    InterProRaw=gene_pred/interproscan/$Organism/$Strain/raw
    $ProgDir/append_interpro.sh $Proteins $InterProRaw
  done
```
## B) SwissProt
<!--
```bash
  screen -a
  qlogin
  ProjDir=/home/groups/harrisonlab/project_files/idris
  cd $ProjDir
  for Proteome in $(ls gene_pred/codingquary/F.*/*/*/final_genes_combined.pep.fasta); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    OutDir=$ProjDir/gene_pred/swissprot/$Species/$Strain/
    mkdir -p $OutDir
    blastp \
    -db /home/groups/harrisonlab/uniprot/swissprot/uniprot_sprot \
    -query $ProjDir/$Proteome \
    -out $OutDir/swissprot_v2015_10_hits.tbl \
    -evalue 1e-100 \
    -outfmt 6 \
    -num_threads 16 \
    -num_alignments 10
  done
``` -->


```bash
for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
  Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
  Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
  OutDir=gene_pred/swissprot/$Organism/$Strain
  SwissDbDir=../../uniprot/swissprot
  SwissDbName=uniprot_sprot
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/swissprot
  qsub $ProgDir/sub_swissprot.sh $Proteome $OutDir $SwissDbDir $SwissDbName
done
```
<!--
```bash
for SwissTable in $(ls gene_pred/swissprot/*/*/swissprot_v2015_10_hits.tbl); do
# SwissTable=gene_pred/swissprot/Fus2/swissprot_v2015_10_hits.tbl
Strain=$(echo $SwissTable | rev | cut -f2 -d '/' | rev)
Organism=$(echo $SwissTable | rev | cut -f3 -d '/' | rev)
echo "$Organism - $Strain"
OutTable=gene_pred/swissprot/$Organism/$Strain/swissprot_v2015_tophit_parsed.tbl
ProgDir=/home/fanron/git_repos/tools/seq_tools/feature_annotation/swissprot
$ProgDir/swissprot_parser.py --blast_tbl $SwissTable --blast_db_fasta ../../../uniprot/swissprot/uniprot_sprot.fasta > $OutTable
done
``` -->
## Effector genes

Putative pathogenicity and effector related genes were identified within Braker
gene models using a number of approaches:

 * A) From Augustus gene models - Identifying secreted proteins
 * B) From Augustus gene models - Effector identification using EffectorP


## A) From Augustus gene models - Identifying secreted proteins

 Required programs:
  * SignalP-4.1
  * TMHMM

 Proteins that were predicted to contain signal peptides were identified using
 the following commands:

 ```bash
  SplitfileDir=/home/fanron/git_repos/tools/seq_tools/feature_annotation/signal_peptides
  ProgDir=/home/fanron/git_repos/tools/seq_tools/feature_annotation/signal_peptides
  CurPath=$PWD
  # for Proteome in $(ls gene_pred/codingquary1/V.*/*/*/final_genes_combined.pep.fasta); do
  for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    SplitDir=gene_pred/final_genes_split/$Organism/$Strain
    mkdir -p $SplitDir
    BaseName="$Organism""_$Strain"_final_preds
    $SplitfileDir/splitfile_500.py --inp_fasta $Proteome --out_dir $SplitDir --out_base $BaseName
    for File in $(ls $SplitDir/*_final_preds_*); do
      Jobs=$(qstat | grep 'pred_sigP' | wc -l)
      while [ $Jobs -gt 20 ]; do
        sleep 10
        printf "."
        Jobs=$(qstat | grep 'pred_sigP' | wc -l)
      done
      printf "\n"
      echo $File
      qsub $ProgDir/pred_sigP.sh $File signalp-4.1
    done
  done
 ```

 The batch files of predicted secreted proteins needed to be combined into a
 single file for each strain. This was done with the following commands:

 ```bash
  for SplitDir in $(ls -d gene_pred/final_genes_split/*/*); do
    Strain=$(echo $SplitDir | rev |cut -d '/' -f1 | rev)
    Organism=$(echo $SplitDir | rev |cut -d '/' -f2 | rev)
    InStringAA=''
    InStringNeg=''
    InStringTab=''
    InStringTxt=''
    SigpDir=final_genes_signalp-4.1
    for GRP in $(ls -l $SplitDir/*_final_preds_*.fa | rev | cut -d '_' -f1 | rev | sort -n); do
      InStringAA="$InStringAA gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_final_preds_$GRP""_sp.aa";
      InStringNeg="$InStringNeg gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_final_preds_$GRP""_sp_neg.aa";
      InStringTab="$InStringTab gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_final_preds_$GRP""_sp.tab";
      InStringTxt="$InStringTxt gene_pred/$SigpDir/$Organism/$Strain/split/"$Organism"_"$Strain"_final_preds_$GRP""_sp.txt";
    done
    cat $InStringAA > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_final_sp.aa
    cat $InStringNeg > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_final_neg_sp.aa
    tail -n +2 -q $InStringTab > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_final_sp.tab
    cat $InStringTxt > gene_pred/$SigpDir/$Organism/$Strain/"$Strain"_final_sp.txt
  done
 ```

 Some proteins that are incorporated into the cell membrane require secretion.
 Therefore proteins with a transmembrane domain are not likely to represent
 cytoplasmic or apoplastic effectors.

 Proteins containing a transmembrane domain were identified:

 ```bash
  # for Proteome in $(ls gene_pred/codingquary1/*/*/*/final_genes_combined.pep.fasta); do
  for Proteome in $(ls gene_pred/final/*/*/final/final_genes_appended_renamed.pep.fasta); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    ProgDir=/home/fanron/git_repos/tools/seq_tools/feature_annotation/transmembrane_helices
    qsub $ProgDir/submit_TMHMM.sh $Proteome
  done
 ```
 Those proteins with transmembrane domains were removed from lists of Signal peptide containing proteins

```bash
  for File in $(ls gene_pred/trans_mem/*/*/*_TM_genes_neg.txt); do
    Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
    echo "$Organism - $Strain"
    TmHeaders=$(echo "$File" | sed 's/neg.txt/neg_headers.txt/g')
    cat $File | cut -f1 > $TmHeaders
    SigP=$(ls gene_pred/final_genes_signalp-4.1/$Organism/$Strain/*_final_sp.aa)
    OutDir=$(dirname $SigP)
    ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_from_fasta.py --fasta $SigP --headers $TmHeaders > $OutDir/"$Strain"_final_sp_no_trans_mem.aa
    cat $OutDir/"$Strain"_final_sp_no_trans_mem.aa | grep '>' | wc -l
  done
```
  V.dahliae - 12008
  943

## B) From Augustus gene models - Effector identification using EffectorP

Required programs:
 * EffectorP.py

```bash
  for Proteome in $(ls gene_pred/codingquary1/*/*/*/final_genes_combined.pep.fasta); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    BaseName="$Organism"_"$Strain"_EffectorP
    OutDir=analysis/effectorP/$Organism/$Strain
    ProgDir=~/git_repos/tools/seq_tools/feature_annotation/fungal_effectors
    qsub $ProgDir/pred_effectorP.sh $Proteome $BaseName $OutDir
  done
```
Those genes that were predicted as secreted and tested positive by effectorP were identified:

```bash
  for File in $(ls analysis/effectorP/*/*/*_EffectorP.txt); do
    Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
    echo "$Organism - $Strain"
    Headers=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_headers.txt/g')
    cat $File | grep 'Effector' | cut -f1 > $Headers
    Secretome=$(ls gene_pred/final_genes_signalp-4.1/$Organism/$Strain/*_final_sp_no_trans_mem.aa)
    OutFile=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted.aa/g')
    ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_from_fasta.py --fasta $Secretome --headers $Headers > $OutFile
    OutFileHeaders=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted_headers.txt/g')
    cat $OutFile | grep '>' | tr -d '>' > $OutFileHeaders
    cat $OutFileHeaders | wc -l
    Gff=$(ls gene_pred/codingquary1/$Organism/$Strain/*/final_genes_appended.gff3)
    EffectorP_Gff=$(echo "$File" | sed 's/_EffectorP.txt/_EffectorP_secreted.gff/g')
    ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_gff_for_sigP_hits.pl $OutFileHeaders $Gff effectorP ID > $EffectorP_Gff
  done
```
V.dahliae - 12008
candidate secreted effectors
194

## C) CAZY proteins

Carbohydrte active enzymes were idnetified using CAZYfollowing recomendations
at http://csbl.bmb.uga.edu/dbCAN/download/readme.txt :

```bash
  for Proteome in $(ls gene_pred/codingquary1/*/*/*/final_genes_combined.pep.fasta); do
    Strain=$(echo $Proteome | rev | cut -f3 -d '/' | rev)
    Organism=$(echo $Proteome | rev | cut -f4 -d '/' | rev)
    OutDir=gene_pred/CAZY/$Organism/$Strain
    mkdir -p $OutDir
    Prefix="$Strain"_CAZY
    CazyHmm=../../../dbCAN/dbCAN-fam-HMMs.txt
    ProgDir=/home/fanron/git_repos/tools/seq_tools/feature_annotation/HMMER
    # ProgDir=/home/gomeza/git_repos/emr_repos/tools/seq_tools/feature_annotation/HMMER
    qsub $ProgDir/sub_hmmscan.sh $CazyHmm $Proteome $Prefix $OutDir
  done
```
The Hmm parser was used to filter hits by an E-value of E1x10-5 or E 1x10-e3 if they had a hit over a length of X %.

Those proteins with a signal peptide were extracted from the list and gff files
representing these proteins made.

```bash
  for File in $(ls gene_pred/CAZY/*/*/*CAZY.out.dm); do
    Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
    Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
    OutDir=$(dirname $File)
    echo "$Organism - $Strain"
    ProgDir=/home/groups/harrisonlab/dbCAN
    $ProgDir/hmmscan-parser.sh $OutDir/12008_CAZY.out.dm > $OutDir/12008_CAZY.out.dm.ps
    SecretedProts=$(ls gene_pred/final_genes_signalp-4.1/$Organism/$Strain/"$Strain"_final_sp_no_trans_mem.aa)
    SecretedHeaders=$(echo $SecretedProts | sed 's/.aa/_headers.txt/g')
    cat $SecretedProts | grep '>' | tr -d '>' > $SecretedHeaders
    Gff=$(ls gene_pred/codingquary1/*/*/*/final_genes_appended.gff3)
    CazyGff=$OutDir/12008_CAZY.gff
    ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
    $ProgDir/extract_gff_for_sigP_hits.pl $SecretedHeaders $Gff CAZyme ID > $CazyGff
  done
```

```bash
for File in $(ls gene_pred/CAZY/*/12008/*CAZY.out.dm); do
      Strain=$(echo $File | rev | cut -f2 -d '/' | rev)
      Organism=$(echo $File | rev | cut -f3 -d '/' | rev)
      OutDir=$(dirname $File)
      echo "$Organism - $Strain"
      ProgDir=/home/groups/harrisonlab/dbCAN
      $ProgDir/hmmscan-parser.sh $OutDir/"$Strain"_CAZY.out.dm > $OutDir/"$Strain"_CAZY.out.dm.ps
      CazyHeaders=$(echo $File | sed 's/.out.dm/_headers.txt/g')
      cat $OutDir/"$Strain"_CAZY.out.dm.ps | cut -f3 | sort | uniq > $CazyHeaders
      echo "number of CAZY genes identified:"
      cat $CazyHeaders | wc -l
      # Gff=$(ls gene_pred/codingquary1/$Organism/$Strain/final/final_genes_appended.gff3)
      Gff=$(ls gene_pred/codingquary1/$Organism/$Strain/final/final_genes_appended.gff3)
      CazyGff=$OutDir/"$Strain"_CAZY.gff
      ProgDir=/home/fanron/git_repos/tools/gene_prediction/ORF_finder
      $ProgDir/extract_gff_for_sigP_hits.pl $CazyHeaders $Gff CAZyme ID > $CazyGff

      SecretedProts=$(ls gene_pred/final_genes_signalp-4.1/$Organism/$Strain/"$Strain"_final_sp_no_trans_mem.aa)
      SecretedHeaders=$(echo $SecretedProts | sed 's/.aa/_headers.txt/g')
      cat $SecretedProts | grep '>' | tr -d '>' > $SecretedHeaders
      CazyGffSecreted=$OutDir/"$Strain"_CAZY_secreted.gff
      $ProgDir/extract_gff_for_sigP_hits.pl $SecretedHeaders $CazyGff Secreted_CAZyme ID > $CazyGffSecreted
      echo "number of Secreted CAZY genes identified:"
      cat $CazyGffSecreted | grep -w 'mRNA' | cut -f9 | tr -d 'ID=' | cut -f1 -d ';' > $OutDir/"$Strain"_CAZY_secreted_headers.txt
      cat $OutDir/"$Strain"_CAZY_secreted_headers.txt | wc -l
    done
```
V.dahliae - 12008
number of CAZY genes identified:
620
number of Secreted CAZY genes identified:
298

Note - the CAZY genes identified may need further filtering based on e value and
cuttoff length - see below:

Cols in yourfile.out.dm.ps:
1. Family HMM
2. HMM length
3. Query ID
4. Query length
5. E-value (how similar to the family HMM)
6. HMM start
7. HMM end
8. Query start
9. Query end
10. Coverage

* For fungi, use E-value < 1e-17 and coverage > 0.45

* The best threshold varies for different CAZyme classes (please see http://www.ncbi.nlm.nih.gov/pmc/articles/PMC4132414/ for details). Basically to annotate GH proteins, one should use a very relax coverage cutoff or the sensitivity will be low (Supplementary Tables S4 and S9); (ii) to annotate CE families a very stringent E-value cutoff and coverage cutoff should be used; otherwise the precision will be very low due to a very high false positive rate (Supplementary Tables S5 and S10)


## D) Identify Small secreted cysteine rich proteins

in secretome:

```bash
  ProgPath=/home/fanron/git_repos/tools/pathogen/sscp
  for Filez in $(ls gene_pred/final_genes_signalp-4.1/V.dahliae/12008/*_sp.aa | grep -v _neg_sp.aa); do            
  echo "$Filez"
  qsub "$ProgPath"/sub_sscp.sh "$Filez"
  done
```

in effectorP:
```bash
  ProgPath=/home/fanron/git_repos/tools/pathogen/sscp
  for Filez in $(ls analysis/effectorP/V.dahliae/12008/*.aa); do            
  echo "$Filez"
  qsub "$ProgPath"/sub1_sscp.sh "$Filez"
  done
```
## E) AntiSMASH

Antismash was run to identify clusters of secondary metabolite genes within
the genome. Antismash was run using the weserver at:
http://antismash.secondarymetabolites.org

The assembly and Gff annotaiton of gene models was converted into EMBL format prior to submission:

<!-- ```bash
for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
OutDir=gene_pred/antismash/$Organism/$Strain
mkdir -p $OutDir
Gff=$(ls gene_pred/codingquary1/V.dahliae/12008/final/final_genes_appended.gff3)
seqret -sequence $Assembly -feature -fformat gff -fopenfile $Gff -osformat embl -auto
mv contig_1_pilon.embl $OutDir/"$Strain"_parsed_genome.embl
done
``` -->
<!--
```bash
for Assembly in $(ls repeat_masked/*/*/*/*_contigs_unmasked.fa); do
Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
echo "$Organism - $Strain"
OutDir=gene_pred/antismash/$Organism/$Strain
mkdir -p $OutDir
Gff=$(ls gene_pred/codingquary1/V.dahliae/12008/final/final_genes_appended.gff3)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/secondary_metabolites
$ProgDir/antismash_gff2tab.py --gff $Gff --fasta $Assembly --out $OutDir
cp $ProgDir/format_embl.py $OutDir/.
# cp $Assembly $OutDir/assembly.fasta
CurDir=$PWD
cd $OutDir
python format_embl.py
cd $CurDir
rm $OutDir/annotationtable.txt
seqret -sequence $Assembly -feature -fformat gff -fopenfile $Gff -osformat embl -auto
mv contig_1_pilon.embl $OutDir/"$Strain"_parsed_genome.embl
done
```
 -->

```bash
AntiSmash=analysis/antismash/79c1471f-4a2b-41f7-ba36-18ba94675f59/contig_1_pilon.final.gbk
OutDir=$(dirname $AntiSmash)
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/feature_annotation/secondary_metabolites
$ProgDir/antismash2gff.py --inp_antismash $AntiSmash > $OutDir/Fus2_secondary_metabolite_regions.gff
GeneGff=gene_pred/final_genes/F.oxysporum_fsp_cepae/Fus2_canu_new/final/final_genes_appended.gff3
bedtools intersect -u -a $GeneGff -b $OutDir/Fus2_secondary_metabolite_regions.gff > $OutDir/metabolite_cluster_genes.gff
cat $OutDir/metabolite_cluster_genes.gff | grep -w 'mRNA' | cut -f9 | cut -f2 -d '=' | cut -f1 -d ';' > $OutDir/metabolite_cluster_gene_headers.txt

```
