# Using Bioinformtics to find integrated viral signatures within genomes
---
There are a variety of tools to finding viral sequences within bacterial genomes, however, these can be inaccurate and reply on only one algorthim. Here we combined VirSorter and VirFinder using viral hallmarks to find viruses and viral k-mer frequencies to confirm they are viral. Lastly we use VContact2 to visualise their taxonomy.

Before you start:
*   Have [VirSorter](https://github.com/simroux/VirSorter) installed
*   Have [VirFinder](https://github.com/jessieren/VirFinder) installed
*   Have [vContact2](https://bitbucket.org/MAVERICLab/vcontact2) installed

Run VirSorter on your sequences with the ViromeDB instead of refseqdb

```
$ cat *.fasta > all_sags.fasta
$ wrapper_phage_contigs_sorter_iPlant.pl -f all_sags.fasta --db 2 --ncpu 16 --data-dir ~/ashley/tools/virsorter/virsorter-data/ --diamond 2>&1 | tee virsorter.log
```

Now run VirFinder on the predicted viral sequences to verify they have viral k-mers

```
$ cd virsorter-out/Predicted_viral_sequences
$ cat VIRSorter_*.fasta > cat123456.fasta
$ Rscript virfinder.R
```
Next we need 2 files for vContact2 to work. A protein translation file and a gene_to_genome file. We can't use the `/virsorter-out/fasta/VIRSorter_prots.faa` file because it contains genes that even VirSorter thinks isn't viral. (In my test, it contain prophage genes 1-72 even though VirSorter said the prophage was between genes 1-25. If we convert nuclotides to proteins, we run the risk of having different protein sequences that VIRSorter has found. Therfore we need to convert the genbank files into proteins and only take the proteins we are interested in. 
```
$ for i in *.gb; do python converter.py $i; done
$ cat *converted* > converted.faa
$ for i in $(cut -f1 -d"," VirFinder_cat123456_trm.csv); do echo $i | sed 's/"//g' | grep -A 1 -f - converted.faa; done > VirFinder_cat123456.faa
```
Now we just need the genome to gene file. We can use `vContact2`'s inbuilt function to do that. Or With a bit of `awk` magic we can convert the .faa file into it. The `None_provided` on the end is meant to be the function of the translated protein sequence and you can add this in if you figure out what the protein translations are first.
```
$ vcontact2_gene2genome -p VirFinder_cat123456.faa -o gene_to_genome.csv -s VIRSorter
$ awk 'BEGIN{OFS=","; print "protein_id","contig_id","keywords"} NR%2==1 {OFS=","; gsub(/>/,"",$1); print $1,$3,"None_provided"}' VirFinder_cat123456.faa > gene_to_genome.csv
```

Now we can run vContact2 on the protein files to file their taxonomy

```
$ vcontact2 --raw-proteins VirFinder_cat123456.faa --proteins-fp gene_to_genome.csv --c1-bin ~/miniconda3/envs/vContact2/bin/ --output-dir vContact2_output --threads 16 2>&1 | tee vcontact2.log
```
