# Lab-Sequence_Quality

Let's get outselves some sequence data to play with. Data from many projects are available at the Sequence Read Archive (SRA) - this is like GenBank for raw high-throughput sequencing data. (Much of the HTS data from published papers gets archived here.)

## Download sequences from SRA

Luckily, the SRA has its own command line toolkit `sra-tools` for retrieving data. Some of these files can be very big, and this lets us directly download files to Poseidon. Plus, with a command line tool, we can use bash scripting to automate retrieval of lots of sequences at once. (More on this later.)

#### Install sra-tools

First, we need to install `sra-tools` in a new conda environment. Let's create one called *sra_get* (or whatever you like!), activate it, and install `sra-tools`.

```conda create --name sra_get```

```conda activate sra_get```

```conda install -c bioconda sra-tools=2.11.0```

#### Request Poseidon resources for downloading

Since we're not downloading a lot right now, we could go ahead and do it on Poseidon as-is. However, it's good HPC manners to request time and resources first. Let's do that interatively with an `srun` request (make sure you've navigated to the folder you cloned from GitHub for this).

```srun -p compute --time=1:00:00 --ntasks-per-node=1 --mem=8gb --pty bash```

Notice how your prompt changed once resources were assigned? `pn045` (or whatever) is the node that's been assigned to you for the next hour.

Now, activate your `sra_get` conda environment. (If you had already activated it, you would have to do so again after your `srun` request to be able to access it now.)

#### Download fastq files based on the run number 

Now, we are ready to start the download using the `fasterq-dump` command. (This is the streamlined update of the original `fastq-dump` command, courtesy of those clever wags at NCBI.)

Use help to see your options:
```fasterq-dump -h```

The flags `-O` and `-split-3` are commonly used. `-O` lets you set the output directory (default is your current working directory).  `--split-3` is specific to **paired-end reads**, and will create two fastq files, one for the forward reads and the other for the reverse read. `--split-3` will check that each forward read has a reverse read mate, and will not download unpaired reads. `--split-reads` will split forward and reverse without running this check. In theory, all the reads **should** be paired in the raw data, but if they aren't then `--split-3` ensures downstream processing will go more smoothly.

Now, let's all download the file with the accession number SRR8281009. (This is a transcriptome run from _Cymothoa exigua_, everybody's favorite fish tongue-replacing parasitic isopod.)

```fasterq-dump --split-3 --verbose -O sra/ SRR8281009```

#### Examine sequence quality

Check out your new `sra` folder. How big is it? The `du` ("Disk Usage") command is handy for this. `-s` makes it give a summary size over all the files in the directory, and `-h` displays it in handy human-friendly M, G, etc (rather than thousands of bytes.)

```du -sh *fastq```

Now, navigate into your `sra` folder. Hopefully you have 2 files ending in `_1.fastq` and `_2.fastq`. These are your forward and reverse reads from the single run you downloaded.






-use a for loop to download all the sequencing runs. I recommend modifying the command we learned above to produce compressed files: e.g. ```fastq-dump --split-files --gzip -O sra/ ${i}``` [${i} is your variable i.e. sra-id]



There are also ways to parallelize the dowload (e.g. https://github.com/rvalieris/parallel-fastq-dump)

-go onto poseidon and navigate to your working folder

-start a tmux session

-start a slurm job: e.g. ```srun -p compute --time=24:00:00 --ntasks-per-node=4 --mem=8gb --pty bash``` *in this case you request 4 cores*

-load anaconda

-activate your conda environment

-conda install **parallel-fastq-dump**

-use a for loop to download all the sequencing runs using parallel-fastq-dump and taking advantage of the 4 cores you requested e.g. ```parallel-fastq-dump --s ${i} --threads 4 -O sra/ --split-files --gzip```


----------


#### Get accession list
Go to [NCBI](http://www.ncbi.nlm.nih.go/v) http://www.ncbi.nlm.nih.gov

![Alt text](/images/sra1.png)


Select "SRA"
![Alt text](/images/sra2.png)


Insert the accession number of the BioProject and "Search"
"Send results to Run selector"
![Alt text](/images/sra3.png)




![Alt text](/images/sra4.png)


Select (or not) runs

Download the Accession List and the Run Info Table 
![Alt text](/images/accesion.png)
![Alt text](/images/list.png)

You can transfer files (e.g. the accession list) from your local machine to poseidon (and vice versa) by using scp ()scp *source* *destination* )
```scp path-to-file username@poseidon.whoi.edu:path-to-destination-folder```

> Modify the fast-dump command you used above to automate your downloads from SRA
