#Weed Resistance Project

Documentation of the analyses on genomic and protein data from the plants *Lolium multiflorum* and *Sorghum bicolor* on the weed resistance project for John.

##Download Lolium multiflorum genome

Download the genome of *Lolium multiflorum* (Italina ryegrass) with the accession number GCA_019182485.1, unzip it and save it as in FASTA format named "GCA_019182485.1_Rabiosa_v.0_genomic.fasta".

```
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/019/182/485/GCA_019182485.1_Rabiosa_v.0/GCA_019182485.1_Rabiosa_v.0_genomic.fna.gz

gunzip GCA_019182485.1_Rabiosa_v.0_genomic.fna.gz

mv GCA_019182485.1_Rabiosa_v.0_genomic.fna.gz GCA_019182485.1_Rabiosa_v.0_genomic.fasta
```

##Run BUSCO with poales_odb10 database

Run BUSCO (Benchmarking Universal Single-Copy Orthologs) analysis on the *Lolium multiflorum* genome using the poales_odb10 database. BUSCO assesses the completeness of a genome assembly by searching for the presence of single-copy orthologous genes that are expected to be present in a specific database. The command is submitted to a job scheduler using the `sbatch` command and the script "busco_poales.sh".

```
sbatch ~/scratch/private/scripts/busco_poales.sh GCA_019182485.1_Rabiosa_v.0_genomic.fasta
```

##Take list of 99 Sorghum gene symbols

Open a text editor to edit a file named "sorghum_gene.id". This file contains a list of 99 gene symbols for *Sorghum bicolor* identified from [Ananda *et al.* 2021](https://doi.org/10.1002/tpg2.20123).

```
nano sorghum_gene.id
```

##Use Entrez Direct to download protein sequences

Use Entrez Direct to download the protein sequences corresponding to the 99 gene symbols in the "sorghum_gene.id" file. The command reads each line of the file using a while loop and creates a separate file for each gene symbol using the "touch" command. Then, for each gene symbol file, it uses the `esearch` command to search the NCBI protein database for the corresponding protein sequence and the `efetch` command to download it in FASTA format. The resulting protein sequence files are saved with the extension ".faa".

```
cat sorghum_gene.id | while read line; do touch $line; done

for file in *;
    do esearch -db protein -query $file | efetch -format fasta > "$file".faa
    done
```

##Select proteins with single isoform

Select the protein sequences that have only one isoform (i.e., a single copy of the gene) among the downloaded *Sorghum bicolor* protein sequences. The command searches each protein sequence file in the directory "sorghum/" for the number of FASTA headers (i.e., the number of protein isoforms) using the `grep` and `wc` commands. If there is only one header, the file is moved to a new directory named "1orth/".

```
for file in sorghum/*; do orth=$(grep '>' $file | wc -l); if [ $orth -eq 1 ]; then mv $file 1orth/; fi; done
```

##Concatenate Lolium BUSCO proteins and generate blast database

Concatenate the protein sequences from the BUSCO analysis of *Lolium multiflorum* from both multi-copy and single-copy genes into a single file named "lolium_busco_prot.faa". The protein sequences are taken from two separate directories named "multi_copy_busco_sequences/" and "single_copy_busco_sequences/". Finally, the `makeblastdb` command is used to generate a blast database in protein format from the concatenated protein sequences.

```
for file in multi_copy_busco_sequences/*; do cat $file >> lolium_busco_prot.faa; done
for file in single_copy_busco_sequences/*; do cat $file >> lolium_busco_prot.faa; done

makeblastdb -in lolium_busco_prot.faa -dbtype prot -out lolium_prot
```

##BLAST Sorghum proteins against Lolium db

Perform a BLAST (Basic Local Alignment Search Tool) analysis of the *Sorghum bicolor* protein sequences against the *Lolium multiflorum* protein database created in the previous step. The `blastp` command is used to search the database with each protein sequence file in the "download/sorghum_prot/" directory using the options `-max_target_seqs 3` to return up to three best hits, `-outfmt 6` to output results in tabular format, `-evalue 1e-3` to set the E-value threshold to 0.001, and `-num_threads 2` to use two threads. The results are saved in a directory named "blast_results/" with a filename based on the gene symbol used to query the database.

```
for file in download/sorghum_prot/*;
    do fileshort=$(basename $file | sed s/".faa"//g)
    blastp -query $file -db lolium_prot -max_target_seqs 3 -outfmt 6 -evalue 1e-3 -num_threads 2 > blast_results/"$fileshort".outfmt6
    done
```

##Remove outputs without hits

Move any BLAST output files that do not have any hits (i.e., zero-size files) to a directory named "no_hits/".

```
find . -size 0 -exec  mv {}  no_hits \;
```

