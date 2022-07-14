.. highlight:: shell

.. role:: bash(code)
   :language: bash

Pipeline
------------



System requirements
>>>>>>>>>>>>>>>>>>>

* Linux/Unix

Preprocess
::::::::::::::::::::::::::::::::::::::::::::::::


Usage:
:: 

   usage: bash m6a_pipeline.sh <samp> <ref(human, mouse)> <workpath> <refpath> <anno_dir>
   <samp> : sample name
   <ref>  : hg38/mm10
   <workpath> : location of where data files are in
   <refpath> : location of where ref_dir are in
   <anno_dir> : location of where anno.gtf is in


m6a_pipeline.sh
::

   #!/bin/bash
   #You should index reference genome first

   samp=$1
   ref=$2
   work_dir=$3
   ref_dir=$4
   anno_dir=$5
   data_dir=$work_dir/01.Raw
   trim_dir=$work_dir/02.Trim
   bam_dir=$work_dir/03.Bam


   # check input param
   if [ $# -lt 4 ]; then
     echo "usage: bash m6a_pipeline.sh <samp> <ref(human, mouse)> <workpath> <refpath> <anno_dir>"
     echo "<samp> : sample name"
     echo "<ref>  : hg38/mm10"
     echo "<workpath> : location of where data files are in"
     echo "<refpath> : location of where ref_dir are in"
     echo "<anno_dir> : location of where anno.gtf is in"
     exit
   fi


   #check STAR index
   if [ -f $ref_dir/$ref.fa ]||[ -f $anno_dir/$ref.utr.chr.gtf ];then
      if [ ! -f $ref_dir/$ref/*.tab ];then
         echo "You need build STAR index first!"
         STAR --runThreadN 8 --runMode genomeGenerate --genomeDir $ref_dir/$ref --genomeFastaFiles $ref_dir/$ref.fa --sjdbGTFfile $anno_dir/$ref.utr.chr.gtf --sjdbOverhang 149
         STAR --runThreadN 8 --runMode genomeGenerate --genomeDir $ref_dir/${ref}_control --genomeFastaFiles $ref_dir/$ref.fa $ref_dir/control.fa --sjdbGTFfile $anno_dir/$ref.gtf --sjdbOverhang 149
      else
         echo "STAR Index has already built"
      fi
   fi



   #Trim_galore
   echo "Step.01--Trim Raw Data and Get Clean Data"

   if [ ! -d $trim_dir ];then
      mkdir $trim_dir
   fi

   trim_galore -j 7 --phred33 --paired --stringency 3 -e 0.1 $data_dir/${samp}.R1.fastq.gz $data_dir/${samp}.R2.fastq.gz --gzip -o $trim_dir > $trim_dir/${samp}.log 2>&1

   echo "Step.01 Done!"


   #STAR mapping
   echo "Step.02--Mapping Trimmed Reads"

   if [ ! -d $bam_dir ];then
      mkdir $bam_dir
   fi

   if [ ! -d $bam_dir/CHRbam ];then
      mkdir $bam_dir/CHRbam
   fi


   STAR --runThreadN 30 --runMode alignReads --genomeDir $ref_dir/$ref --readFilesCommand zcat --readFilesIn $trim_dir/${samp}.R1_val_1.fq.gz $trim_dir/${samp}.R2_val_2.fq.gz --outFileNamePrefix $bam_dir/${samp}_ --outSAMtype BAM SortedByCoordinate  --outSAMstrandField intronMotif --outFilterMismatchNmax 999 --outFilterMismatchNoverLmax 0.04
   STAR --runThreadN 30 --runMode alignReads --genomeDir $ref_dir/${ref}_control --readFilesCommand zcat --readFilesIn $trim_dir/${samp}.R1_val_1.fq.gz $trim_dir/${samp}.R2_val_2.fq.gz  --outFileNamePrefix $bam_dir/${samp}_cont --outSAMtype BAM SortedByCoordinate  --outSAMstrandField intronMotif --outFilterMismatchNmax 999 --outFilterMismatchNoverLmax 0.04

   /home1/changzhanhe/basic_code/bam_add_chr.sh $bam_dir/${samp}_Aligned.sortedByCoord.out.bam $bam_dir/${samp}.CHR.bam
   mv $bam_dir/${samp}.CHR.bam $bam_dir/CHRbam/


   echo "Step.02 Done!"


   ##Flagstat
   echo "Step.03--Flagstat Bam Files"

   samtools flagstat $bam_dir/${samp}_contAligned.sortedByCoord.out.bam > $bam_dir/${samp}.map.flag.txt

   echo "Step.03 Done!"


   ##IP effiecncy
   echo "Step.04--Mapping Efficiency"

   if [ ! -f $bam_dir/gluc_cluc_reads_count.txt ];then
      awk 'BEGIN{OFS="\t";print "sample","gluc","cluc","gluc/cluc"}' > $bam_dir/gluc_cluc_reads_count.txt
   fi

   gluc=`bamtools count -in $bam_dir/${samp}_contAligned.sortedByCoord.out.bam -region Modified_Control_RNA`
   cluc=`bamtools count -in $bam_dir/${samp}_contAligned.sortedByCoord.out.bam -region Unmodified_Control_RNA`
   awk 'BEGIN{OFS="\t";print "'$samp'","'$gluc'","'$cluc'",'$gluc'/'$cluc'}' >> $bam_dir/gluc_cluc_reads_count.txt


   echo "Step.04 Done!"

   rm $bam_dir/${samp}_contAligned.sortedByCoord.out.bam


   ##Decay
   echo "Step.05--Decay condition"
   java -jar /home1/xuxc/02.bin/picard.jar CollectRnaSeqMetrics I=$bam_dir/${samp}_Aligned.sortedByCoord.out.bam \
      O=$bam_dir/${samp}.RNA_Metrics \
      REF_FLAT=$anno_dir/refFlat.txt \
      CHART_OUTPUT=$bam_dir/${samp}.pdf \
      STRAND_SPECIFICITY=SECOND_READ_TRANSCRIPTION_STRAND

   echo "Step.05 Done!"













