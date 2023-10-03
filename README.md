# Lab-Sequence_Quality

Let's get ourselves some sequence data to play with. Data from many projects are available at the Sequence Read Archive (SRA) - this is like GenBank for raw high-throughput sequencing data. (Much of the HTS data from published papers gets archived here.)

## Download sequences from SRA

Luckily, the SRA has its own command line toolkit `sra-tools` for retrieving data. This lets us directly download files to Poseidon, which is handy as some of these files can be big. Plus, with a command line tool, we can use bash scripting to automate retrieval of lots of sequences at once. (More on this later.)

#### Install sra-tools

First, we need to install `sra-tools` in a new conda environment. Let's create one called *sra_get* (or whatever you like!), activate it, and install `sra-tools`.

```
conda create --name sra_get
conda activate sra_get
conda install -c bioconda sra-tools
```

##### Using `mamba` to speed up downloads
If this is taking a long time, a great alternative is `mamba`. `mamba` is a reimplementation of `conda` (so it does the exact same thing as `conda`, it just uses a different route to do so). Using a magic combination of _parallelization_, `C++`, and much faster solving using an improved algorithm (`libsolv`), `mamba` resolves all of our package installation woes. Most packages should be quick to install with `mamba`. 

Sadly, `mamba` only became available out-of-the-box in `anaconda` versions from 2022.05 and later, so because our Poseidon module as of 10/2/2023 is from 2021, we have to install `mamba` before we can use it. So the steps will be:

```
conda create -n sra_get -c conda-forge -y mamba=1.5.1
conda activate sra_get
mamba install -c bioconda -y sra-tools
```

The `-y` flag says "yes" to `mamba` instead of asking us if downloads are okay. If `mamba` was already installed, we could:

```
mamba create --name sra_get
mamba activate sra_get
mamba install -c bioconda sra-tools
```

just like with `conda`; `mamba` works _exactly_ like `conda` does!

![Speedup gif from https://blog.hpc.qmul.ac.uk/mamba.html](https://github.com/2023-environmental-bioinformatics/Lab-Sequence_Quality/blob/main/libmamba_classic_comparison.gif)

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

Check out your new `sra` folder. How big is it? The `du` ("Disk Usage") command is handy for this. `-s` makes it give a summary size over all the files in the directory, and `-h` displays it in handy human-friendly M, G, etc (rather than thousands of bytes.)

`du -sh sra`

Now, navigate into your `sra` folder. Hopefully you have 2 files ending in `_1.fastq` and `_2.fastq`. These are your forward and reverse reads from the single run you downloaded.


## Examine sequence quality

All right, let's set up a new environment to do some quality control on our shiny new parasite sequence data. There are many ways to quality-trim data. I like Trim Galore!, which we'll use here. Another popular option is Trimmomatic.

```
conda create --name qc_trim
conda activate qc_trim
conda install -c bioconda trim-galore
```

The Trim Galore! conda install helpfully includes FastQC, a program which help you look at various quality parameters of your data. First, let's run it on our raw sequence data:

```
cd sra
fastqc *fastq
```

When it's done, take a look at your folder again. You should now see a `.zip` and a `.html` file for each of your forward and reverse reads. Open up a *local* terminal, navigate to a folder you're using for class stuff, and copy these files over (necessary so we can view the data summaries).

```
scp USERNAME@poseidon.whoi.edu:/[PATH_TO_SRA_FOLDER]/*zip .
scp USERNAME@poseidon.whoi.edu:/[PATH_TO_SRA_FOLDER]/*html .
```

Now, on your own computer, navigate to those files and open the one ending in `*.html`. This should open a webpage with a bunch of summary graphs. (This will open whether or not you're online - it is coded in html, but it's just rendering everything in the `*.zip` folder in a pretty, user-friendly format.)

Let's clean up after ourselves on the HPC. Your `srun` session is probably still running, even though we're done with it. You can check to see what jobs you're running using the `squeue` command:

`squeue -u USERNAME`

Cancel the session and free up the space. If you want to cancel all the jobs you're currently running (right now, probably just this `srun` job), you can do it this way:

`scancel -u USERNAME`

If you're running a bunch of jobs and only want to cancel one of them, find its `JOBID` using `squeue` and cancel it specifically:

`scancel JOBID`

## Trim sequences and check quality again

These sequences look OK, but we definitely have some low-quality (Phred < 20) bases. Let's get rid of them, and while we're at it, let's remove any stray adaptor sequence as well. This **should** get removed as part of the sequencing process, before reads even make it to the user, but the process isn't always perfect - especially for older sequences.

For this, rather than requesting interactive `srun` resources, we're going to run our code in a slurm script that we submit to Poseidon. This requires embedding your code in a file with a slurm header that tells the system what you're requesting, who's requesting it, etc. The great thing about submitting scripts this way is that we don't need to hang around and wait for them to finish (or start, if they are big jobs!). Our job goes in a queue with everyone else's, and when it's our turn, it runs using the time and resources we request. In Poseidon, copy the `slurm_header.txt` script and give it a more descriptive name:

`cp slurm_header.txt isopod_qc.txt`

Open up your new slurm file. Modify it as needed - at a minimum, you need to give it a name (twice!), and enter your email address. You should also tailor the resource request to your needs. For today, set the following:

```
#SBATCH --mem=5G
#SBATCH --time=02:00:00
```

Now, put the commands you want to run **under** the header.

```
trim_galore -q 20 --length 60 --paired --fastqc SRR8281009_1.fastq SRR8281009_2.fastq
```
This is a pretty typical command for a program run via the command line. First is the name of the program - this is case- and punctuation-sensitive. Then a bunch of flags to pass various parameters to the program. Some flags take additional arguments that you define (`-q 20`), and others are just essentially on/off switches for certain behaviors (`--paired`). Sometimes flags will start with two dashes (`--paired`), and sometimes with one (`-q 20`). This varies from program to program, but usually the double-dashed flags spell out a more complete command, while the single-dashed commands are a shortened (often single-letter) abbreviation. Use the `--help` option - and the program's documentation - liberally to keep all of this straight.

In this case, we're telling Trim Galore! that we want to trim bases off the ends of the sequences if they have Phred < 20 (`-q 20`), to drop sequences altogether if they are < 60 bp long after trimming (`--length 60`), that it should expect paired-end reads (`--paired`), and that you want to run FastQC on the cleaned reads (`--fastqc`). Finally, you're telling the program, **in order**, the forward and reverse read files you want processed. There are a lot of other parameters you can set, too - tune these settings to your project's needs. For example, these reads are 75 bp long, so 60 bp seems reasonable for a minimum post-cleaning length. You'd probably want to change this if your reads are, say 50 bp or 150 bp. 

Submit the script via slurm:

`sbatch isopod_qc.txt`

Check to see if it's running. Since you submitted the script to the queue, there's no output to the terminal. Where do you think this is going (is it going anywhere? into the void?)?

Check your email - you should get emailed when you script starts running (not necessarily immediately! large requests may sit in the queue for a while), and when it's ended. The ending emails are often quite useful - they include basic info on **how** the script ended (COMPLETED, TIMED_OUT, FAILED), and **how long** they took to run. This can be very helpful when you're trying to figure out what kind of resources to request for a really big job, especially if it involves doing the same thing to a bunch of samples.

Now take a look at the files in this folder. What's changed?

Open up your `*.log` file.

Now, move your new `*.zip` and `*.html` files over to your local computer, and take a look.


> Suppose you want to retrieve and clean a bunch of samples, and you have a list of sample numbers (`SRA_list.txt` in this repo). Write a script (or two) to automate this process, and embed it / them in a slurm script to submit.


----------

## More SRA details

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

#### Parallelizing sample downloads

In addition to looping `fasterq-dump` to download multiple samples, there are also ways to parallelize the dowload (e.g. https://github.com/rvalieris/parallel-fastq-dump)

-go onto poseidon and navigate to your working folder

-start a tmux session

-start a slurm job: e.g. ```srun -p compute --time=24:00:00 --ntasks-per-node=4 --mem=8gb --pty bash``` *in this case you request 4 cores*

-load anaconda

-activate your conda environment

-conda install **parallel-fastq-dump**

-use a for loop to download all the sequencing runs using parallel-fastq-dump and taking advantage of the 4 cores you requested e.g. ```parallel-fastq-dump --s ${i} --threads 4 -O sra/ --split-files --gzip```
