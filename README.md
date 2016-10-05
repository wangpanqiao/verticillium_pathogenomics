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

<!-- After investigation it was found that contig_17 should be split.

```bash
  ProgDir=~/git_repos/tools/seq_tools/assemblers/assembly_qc/remove_contaminants
  touch tmp.csv
  printf "contig_17\tmanual edit\tsplit\t780978\t780978\tcanu:missassembly\n" > tmp.csv
  for Assembly in $(ls assembly/canu/F.oxysporum_fsp_cepae/Fus2/filtered_contigs/Fus2_canu_contigs_renamed.fasta); do
    Organism=$(echo $Assembly | rev | cut -f4 -d '/' | rev)  
    Strain=$(echo $Assembly | rev | cut -f3 -d '/' | rev)
    OutDir=assembly/canu/$Organism/$Strain/edited_contigs
    mkdir -p $OutDir
    $ProgDir/remove_contaminants.py --inp $Assembly --out $OutDir/"$Strain"_canu_contigs_modified.fasta --coord_file tmp.csv
  done
  rm tmp.csv
``` -->
 
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


Checking PacBio coverage against Spades assembly

```bash
  Assembly=assembly/merged_canu_spades/V.dahliae/12008/filtered_contigs/12008_contigs_renamed.fasta
  Reads=raw_dna/pacbio/V.dahliae/12008/extracted/concatenated_pacbio.fastq
  OutDir=analysis/genome_alignment/bwa/Verticillium/12008/vs_12008
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
  for BestAss in $(ls assembly/merged_canu_spades/*/*/filtered_contigs/12008_contigs_renamed.fasta); do
    Organism=$(echo $BestAss | rev | cut -d "/" -f4 | rev)
    Strain=$(echo $BestAss | rev | cut -d "/" -f3 | rev)
    OutDir=repeat_masked/$Organism/$Strain/filtered_contigs_repmask
    qsub $ProgDir/rep_modeling.sh $BestAss $OutDir
    qsub $ProgDir/transposonPSI.sh $BestAss $OutDir
  done
```

The number of bases masked by transposonPSI and Repeatmasker were summarised
using the following commands:

```bash
  for RepDir in $(ls -d repeat_masked/V.*/*/filtered_contigs_repmask | grep '12008'); do
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
  V.dahliae       12008
  The number of bases masked by RepeatMasker:     3014429
  The number of bases masked by TransposonPSI:    859780
  The total number of masked bases are:   3161584

```bash
  for File in $(ls repeat_masked/*/*/filtered_contigs_repmask/*_contigs_softmasked.fa); do
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
  for Genome in $(ls repeat_masked/V.*/*/*/*_contigs_softmasked.fa); do
    echo $Genome;
    qsub $ProgDir/sub_cegma.sh $Genome dna;
  done
```
** Number of cegma genes present and complete: 95.56%
** Number of cegma genes present and partial: 98.39%

```bash
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/cegma
  cd /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics
  for Genome in $(ls repeat_masked/V.*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    echo $Genome;
    qsub $ProgDir/sub_cegma.sh $Genome dna;
  done
```
** Number of cegma genes present and complete: 95.56%
** Number of cegma genes present and partial: 98.39%

Outputs were summarised using the commands:
```bash
  for File in $(ls gene_pred/cegma/V.*/12008/*_dna_cegma.completeness_report); do
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
There are 237 complete and 7 partial core eukaryotic genes (out of total 248 genes) present in my assembly

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
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    for RNADir in $(ls -d qc_rna/paired/V.*/12008PDA); do
      Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
      echo "$Timepoint"
      FileF=$(ls $RNADir/F/*_trim.fq.gz)
      FileR=$(ls $RNADir/R/*_trim.fq.gz)
      OutDir=alignment/$Organism/$Strain/$Timepoint
      ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
      qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir
    done
  done
```
81.5% overall read mapping rate.
72.8% concordant pair alignment rate.


```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    echo "$Organism - $Strain"
    for RNADir in $(ls -d qc_rna/paired/V.*/12008CD); do
      Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
      echo "$Timepoint"
      FileF=$(ls $RNADir/F/*_trim.fq.gz)
      FileR=$(ls $RNADir/R/*_trim.fq.gz)
      OutDir=alignment/$Organism/$Strain/$Timepoint
      ProgDir=/home/fanron/git_repos/tools/seq_tools/RNAseq
      qsub $ProgDir/tophat_alignment.sh $Assembly $FileF $FileR $OutDir
    done
  done
```
80.4% overall read mapping rate.
70.9% concordant pair alignment rate.

lignments were concatenated prior to running cufflinks:
Cufflinks was run to produce the fragment length and stdev statistics:

if run the commands in a node other than cluster, using the script:

```bash
qlogin -pe smp 8 -l virtual_free=1G
```
```bash
for Assembly in $(ls /home/groups/harrisonlab/project_files/verticillium_dahliae/pathogenomics/repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
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
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  for AcceptedHits in $(ls alignment/$Organism/$Strain/*/accepted_hits.bam); do
  Timepoint=$(echo $AcceptedHits | rev | cut -f2 -d '/' | rev)
  echo $Timepoint
  OutDir=gene_pred/cufflinks/$Organism/$Strain/"$Timepoint"_prelim
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
    -127bp 233-(180*2) where 180? was the mean read length. This was provided to tophat
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
    -125bp 225-(175*2) where 175? was the mean read length. This was provided to tophat
    on a second run (as the -r option) along with the fragment length stdev to
    increase the accuracy of mapping.


Then Rnaseq data was aligned to each genome assembly:

```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
  Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
  Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
  echo "$Organism - $Strain"
  for RNADir in $(ls -d qc_rna/paired/*/12008PDA | grep -v -e '_rep'); do
  Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
  echo "$Timepoint"
  FileF=$(ls $RNADir/F/*_trim.fq.gz)
  FileR=$(ls $RNADir/R/*_trim.fq.gz)
  OutDir=alignment/$Organism/$Strain/$Timepoint
  InsertGap='-127'
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
  Timepoint=$(echo $RNADir | rev | cut -f1 -d '/' | rev)
  echo "$Timepoint"
  FileF=$(ls $RNADir/F/*_trim.fq.gz)
  FileR=$(ls $RNADir/R/*_trim.fq.gz)
  OutDir=alignment/$Organism/$Strain/$Timepoint
  InsertGap='-125'
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
  70.9% concordant pair alignment rate.

#### Braker prediction

Before braker predictiction was performed, I double checked that I had the genemark key in my user area and copied it over from the genemark install directory:

```bash
ls ~/.gm_key
cp /home/armita/prog/genemark/gm_key_64 ~/.gm_key
```
```bash
    #for Assembly in $(ls repeat_masked/N.ditissima/R0905_pacbio_canu/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
    #Jobs=$(qstat | grep 'tophat' | grep -w 'r' | wc -l)
    #while [ $Jobs -gt 1 ]; do
    #sleep 10
    #printf "."
    #Jobs=$(qstat | grep 'tophat' | grep -w 'r' | wc -l)
    #done
    #printf "\n"
    #Strain=$(echo $Assembly| rev | cut -d '/' -f3 | rev)
    #Organism=$(echo $Assembly | rev | cut -d '/' -f4 | rev)
    #echo "$Organism - $Strain"
    #mkdir -p alignment/$Organism/$Strain/concatenated
    #samtools merge -f alignment/$Organism/$Strain/concatenated/concatenated.bam \
    #alignment/$Organism/R0905_pacbio_canu/R0905/accepted_hits.bam
    #OutDir=gene_pred/braker/$Organism/"$Strain"_braker_first
    #AcceptedHits=alignment/$Organism/$Strain/concatenated/concatenated.bam
    #GeneModelName="$Organism"_"$Strain"_braker_first
    #rm -r /home/gomeza/prog/augustus-3.1/config/species/"$Organism"_"$Strain"_braker_first
    #ProgDir=/home/gomeza/git_repos/emr_repos/tools/gene_prediction/braker1
    #qsub $ProgDir/sub_braker_fungi.sh $Assembly $OutDir $AcceptedHits $GeneModelName
    #done
```

Braker predictiction was performed using softmasked genome, not unmasked one.

```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
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
  OutDir=gene_pred/braker/$Organism/"$Strain"_PDA_braker_fourth
  AcceptedHits=alignment/repeat_masked/12008PDA_accurate/accepted_hits.bam
  GeneModelName="$Organism"_"$Strain"_braker_fourth
  rm -r /home/gomeza/prog/augustus-3.1/config/species/"$Organism"_"$Strain"_braker_fourth
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/braker1
  qsub $ProgDir/sub_braker_fungi.sh $Assembly $OutDir $AcceptedHits $GeneModelName
  done
```


```bash
  for Assembly in $(ls repeat_masked/*/*/*/*_contigs_softmasked_repeatmasker_TPSI_appended.fa); do
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
  OutDir=gene_pred/braker/$Organism/"$Strain"_CD_braker_fourth
  AcceptedHits=alignment/repeat_masked/12008CD_accurate/accepted_hits.bam
  GeneModelName="$Organism"_"$Strain"_braker_fourth_CD
  rm -r /home/gomeza/prog/augustus-3.1/config/species/"$Organism"_"$Strain"_braker_fourth_CD
  ProgDir=/home/fanron/git_repos/tools/gene_prediction/braker1
  qsub $ProgDir/sub_braker_fungi.sh $Assembly $OutDir $AcceptedHits $GeneModelName
  done
```

Fasta and gff files were extracted from Braker1 output.

```bash
  for File in $(ls gene_pred/braker/V.dahliae/12008_PDA_braker_fourth/*/augustus.gff); do
  getAnnoFasta.pl $File
  OutDir=$(dirname $File)
  echo "##gff-version 3" > $OutDir/augustus_extracted.gff
  cat $File | grep -v '#' >> $OutDir/augustus_extracted.gff
  done
```

The relationship between gene models and aligned reads was investigated. To do
this aligned reads needed to be sorted and indexed:

  Note - IGV was used to view aligned reads against the Fus2 genome on my local
  machine.

```bash
#InBam=alignment/N.ditissima/R0905_pacbio_canu/concatenated/concatenated.bam
#ViewBam=alignment/N.ditissima/R0905_pacbio_canu/concatenated/concatenated_view.bam
#SortBam=alignment/N.ditissima/R0905_pacbio_canu/concatenated/concatenated_sorted
#samtools view -b $InBam > $ViewBam
#samtools sort $ViewBam $SortBam
#samtools index $SortBam.bam
```