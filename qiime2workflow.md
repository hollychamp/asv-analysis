**Step 1: Install miniconda**

```
mkdir -p ~/miniconda3 
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 
-O ~/miniconda3/miniconda.sh 
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3 
rm -rf ~/miniconda3/miniconda.sh 
~/miniconda3/bin/conda init bash 
~/miniconda3/bin/conda init zsh
```
You may need to install wget using 
```
apt install wget
```
Then close and reopen the terminal and activate miniconda
```
conda activate
```

**Step 2: Install mamba(miniforge)**

```
wget -O Miniforge3.sh "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh" 
conda install mamba -c conda-forge
```
After downloaded and pressed y to proceed 
```
mamba init
```
Then close and reopen terminal again 

**Step 3: Installing q2-fondue**

```
mamba create -y -n fondue \ 
   -c https://packages.qiime2.org/qiime2/2023.2/tested/ \ 
   -c conda-forge -c bioconda -c defaults \ 
   q2cli q2-fondue 
mamba activate fondue  
qiime dev refresh-cache 
```
If fondue has been installed correctly the following command will bring up the fondue manual

```
qiime fondue --help
```

Then you need to look at the configuration of the sra-toolkit 

```
vdb-config -i
```
An interface will pop up and you just need to press x 

**Step 4: Importing the data**

Import the metadata sheet from the SRA run selector into google sheets and remove every column except the run column. Rename this column to 'id' and save it as a .tsv file. 

Copy this file into your docker container by first moving out of your docker container
```
docker cp /users/(name)/(folder_name)/(file_name) (container_id):/(to_the_place_you_want_the_file_to_be) 
```
Then enter back into your docker container and activate fondue 
```
mamba activate fondue
```
Then change your .tsv file into a qiime2 artifact (.qza file)
```
qiime tools import \
  --type NCBIAccessionIDs \
  --input-path runs.tsv \
  --output-path runs.qza
```
Then qiime2 will use this file to import every sequence read needed directly from SRA
```
qiime fondue get-sequences \ 
      --i-accession-ids runs.qza \ 
      --p-email your_email@somewhere.com \ 
      --o-single-reads single_reads.qza \ 
      --o-paired-reads paired_reads.qza \ 
      --o-failed-runs failed_ids.qza \ 
      --verbose
```
You can now exit q2-fondue
```
mamba activate base
```

**Step 5: Make a summary table of your reads so you know what parameters neeeded when trimming**
```
qiime demux summarize --i-data paired_reads.qza --o-visualization demuxsequence-summary.qzv
```

**Step 6: Trimming reads**

Use the dada2 programme to trim reads, for this we will be trimming according to quality and assuming that primers have already been excised from the sequences. Other people may look at the summary table made in the previous step and choose to excise based on when the quality scores start to drop and therefore use the --p-trunc-len lines to do this. 

```
qiime dada2 denoise-paired\ 
  --i-demultiplexed-seqs paired_reads.qza \ 
  --p-trunc-q 20 \ 
  --p-trunc-len-f 0 \ 
  --p-trunc-len-r 0 \ 
  --o-representative-sequences rep-seqs.qza \ 
  --o-table table.qza \ 
  --o-denoising-stats stats.qza
```

**Step 7: Create an additional metadata file**

Create a new metadata file by removing all columns from the ncbi sourced metadata sheet. You need to have a file that just has the run names in one column and the group the sample belongs to in another column. Make sure to save this as a .tsv file and move it into docker. 

**Step 8: Creating a feature table**
Create a feature table that can be used for further analysis 
```
qiime feature-table summarize --i-table table.qza \ 
  --o-visualization table.qzv \ 
  --m-sample-metadata-file metadata.tsv \ 
qiime feature-table tabulate-seqs \ 
 	--i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv
```
**Step 10: Make a phylogenetic tree for further analysis**

This step is so that when doing further analysis, you know what species/genus relates to each amplicon/your 'paired reads'.  
```
qiime phylogeny align-to-tree-mafft-fasttree \
   --i-sequences rep-seq.qza \
   --o-alignment aligned-rep-seq.qza \
   --o-masked-alignment masked-aligned-rep-seqs.qza \
   --o-tree unrooted-tree.qza \
   --o-rooted-tree rooted-tree.qza
```


**Step 9: Further analysis**

Now you can perform any diversity measures/analysis you find suitable (a quick google will bring up the coding you will need)
