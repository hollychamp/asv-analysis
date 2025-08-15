**Step 1: Create your manifest and metadata files**

Open the metadata file of the study you want to use. Also, open up a blank spreadsheet on google sheets. You will need to make two files – one is a manifest file (used to import your sequences into qiime) and one is a metadata file (used to know which sample is in which group). 

First, we will make the manifest file. For this you will need to create a file where you list all the run codes that you will use under the heading – ‘sample-id’ as well as their forward and reverse file paths (under the heading forward-absolute-filepath and reverse-absolute-filepath).  put our files into /mainfs/scratch/username/diss. Also, the file names for the forward reads for each file will be in this format: ERR599353_1_unpaired.fastq.gz and the reverse reads will be in this format: ERR599353_2_unpaired.fastq.gz.

![Manifest Example](images/manifestfile_example.png)

You will then need to export this file as a **.tsv** file.

Then you will need to create your metadata file. This will be where you have a list of your sample-ids and the group they belong to. Again, you will do this in google sheets and export as a **.tsv** file. The file should have two columns: 'sample-id' and 'group'.

![Metadata Example](images/metadatafile_example.png)

**Step 2: Download the reads**

We will use fastq-dl to download our reads from NCBI SRA. First we will need to install fastq-dl into a new conda environment.
>You will need to activate and initialise conda on your HPC account if you have not already done so. You can find details of this on the Iridis wiki on sharepoint. 
```
conda create -n fastq-dl -c conda-forge -c bioconda fastq-dl

conda activate fastq-dl
```
Then you will need to find the SRA project number and insert it into the following command.
```
fastq-dl -a PREJEB23224
```

Once your sequences have downloaded, refresh your file viewer and delete all the files that you are not going to use.

**Step 3: Uploading files**

Upload both your manifest file and your metadata file to your scratch directory using MobaXTerm's file viewer. 

**Step 4: Importing your sequences into a qiime artifact**

First, activate your qiime environment. 
>If you have not already made a conda environment for qiime2 - follow instructions on https://docs.qiime2.org/2024.10/install/native/ 
```
conda activate qiime2-amplicon-2024.10
```
Then you will need to specify your manifest file, what you want file name you want to output as well as the type of data and the input-format
>For single-end reads, --type = 'SampleData[SequencesWithQuality]' and --input-format = SingleEndFastqManifestPhred33V2
>For paired-end reads, --type = 'SampleData[PairedEndSequencesWithQuality]' and input-format = PairedEndFastqManifestPhred33V2
>Note for both of the above it specifies that the file will use Phred33 scoring - if this command does not work - change input to Phred64
```
qiime tools import --type 'SampleData[SequencesWithQuality]' --input-path beemanifest.tsv --output-path bee.qza --input-format SingleEndFastqManifestPhred33V2
```
**Step 5: Visualising the quality of your sequences**

First, we need to convert our sequences into a visual quality plot. In the code below please specify your .qza file created in the previous step and what you want your visualisation file to be called. 
```
qiime demux summarize --i-data bee.qza --o-visualization bee-summary.qzv
```
Then we need to download this file onto our desktop and view it on qiime2 view. Note where sequence quality drops below ~20 so we know where to truncate our reads. 

**Step 6: Removing any adapters**

For this step and subsequent steps we may need to create a script, to send jobs that require more resources to compute nodes of the HPC. You can do this by opening up a text editor and copy and pasting the script format detailed below. Every time a code block begins with #!/bin/bash, you will need to make it into a script. 
>In the script below you will need to alter the --i-demultiplexed-sequences to your .qza file and change the --o-trimmed-sequences to what you would like to name the output. You will also need to change trim-single to trim-paired if you are using paired end sequences. 
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --time=0:10:00
#SBATCH --mem=5GB

qiime cutadapt trim-single --i-demultiplexed-sequences bee.qza --o-trimmed-sequences bee-trimmed.qza
```
You can run a script using sbatch scriptname.sh. Make sure your scripts have finished running and are successful before moving on! 

**Step 7: Quality filtering**

This step will filter out poor-quality reads as well as make our feature table!
>In this script you will need to alter denoise-single to denoise-paired if you are using paired-end reads.
>You will also need to change --i-demultiplexed-seqs to your trimmed.qza file, --o-representative-sequences, --o-table and --o-denoising stats to the names of your output files you want.
>Also change--p-trunc-len to the base number where quality falls below ~20 (right-hand side of quality plot - if you have poor quality at the start of your reads you may need to use --trimLeft also)
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --time=60:00:00
#SBATCH --mem=30GB
#SBATCH --cpus-per-task=30
qiime dada2 denoise-single \
  --i-demultiplexed-seqs bee-trimmed.qza \
  --p-trunc-len 223 \
  --o-representative-sequences bee-rep-seqs.qza \
  --o-table bee-table.qza \
  --o-denoising-stats bee-stats.qza \
  --p-n-threads 30\
```
**Step 8: Analyse percentage of remaining reads**

Next we need to look at how many reads were removed in the previous step. To fo this change the --m-input-file to your denoising stats file from the previous step. Change --o-visualization to file name needed.

```
qiime metadata tabulate \
  --m-input-file bee-stats.qza \
  --o-visualization bee-stats.qzv
```
Then view the .qzv file using qiime2 view. IF <30-40% of the reads remain you will need to go back to the previous step and look at changing the quality settings. 

**Step 9: Visualising your feature table**

We need to visualise our feature table in order to see which samples have low sequencing depth so they can be removed. 
First we need to create a visualisation of our feature table and the view it in qiime2 view. 
> Change --i-table your featuretable.qza file and change --o-visualization to the output name you want.
```
qiime feature-table summarize --i-table bee-table.qza \
--o-visualization bee-table.qzv \
--m-sample-metadata-file bee-metadata.tsv
```
Deciding on the cut-off sampling depth value is somewhat situational but really you want to keep as many samples as possible but with as high a sampling depth as possible. 

**Step 10: Generating a phylogenetic tree**

>In this script you will need to change the --i-sequences and change output names to your choosing.
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --mem=3GB
#SBATCH --time=01:00:00
#SBATCH --ntasks-per-core=1

qiime phylogeny align-to-tree-mafft-fasttree --i-sequences bee-rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza
```
**Step 11: Classify your features**

Here we assign each of our features to taxa in order to explore the microbial composition of each sample. 
>In this script you will need to change the --i-classifier to the one you have downloaded (can be downloaded on the qiime website)
>As well as your --i-reads rep-seqs.qza file and the output name if you wish
```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --mem=1GB
#SBATCH --time=0:10:00
#SBATCH --ntasks-per-core=1

qiime feature-classifier classify-sklearn --i-classifier gg-13-8-99-515-806-nb-classifier.qza --i-reads bee-rep-seqs.qza --o-classification taxonomy.qza
```

**Step 12: Calculate core diversity metrics**
Qiime has a very handy built-in command that calculates a range of common alpha- and beta-diversity indices in one go
>In this script you will need to change the --i-phylogeny to your rooted-tree.qza file, --i-table to your feature table file and --m-metadata-file to your metadata file made in the beginning
>You will also need to change the sampling depth to the number you chose in step 9.

```
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --mem=1GB
#SBATCH --time=0:10:00
#SBATCH --ntasks-per-core=1

qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza --i-table bee-table.qza --p-sampling-depth 9999 --m-metadata-file bee_metadata.tsv --output-dir core-metrics-results
```

