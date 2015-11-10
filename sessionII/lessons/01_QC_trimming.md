
---
title: "NGS workflow - Quality Control - Trimming"
author: "Bob Freeman, Mary Piper"
date: "Tuesday, November 10, 2015"
---

Approximate time: 60 minutes

## Learning Objectives:
* Use Trimmommatic to clean FastQ reads
* Use a For loop to automate operations on multiple files

##Quality Control - Trimming

![Workflow](../img/rnaseq_workflow_trimming.png)

###How to clean reads using *Trimmomatic*

Once we have an idea of the quality of our raw data, it is time to trim away adapters and filter out poor quality score reads. To accomplish this task we will use [*Trimmomatic*](http://www.usadellab.org/cms/?page=trimmomatic).

*Trimmomatic* is a java based program that can remove sequencer specific reads and nucleotides that fall below a certain threshold. *Trimmomatic* can be multithreaded to run quickly. 

Let's load the *Trimmomatic* module:

`$ module load seq/Trimmomatic/0.33`

By loading the *Trimmomatic* module, the **trimmomatic-0.33.jar** file is now accessible to us in the **opt/** directory, allowing us to run the program. 

Because *Trimmomatic* is java based, it is run using the command:

`$ java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar`



What follows below are the specific commands that tells the *Trimmomatic* program exactly how you want it to operate. *Trimmomatic* has a variety of options and parameters:

* **_-threads_** How many processors do you want *Trimmomatic* to run with?
* **_SE_** or **_PE_** Single End or Paired End reads?
* **_-phred33_** or **_-phred64_** Which quality score do your reads have?
* **_SLIDINGWINDOW_** Perform sliding window trimming, cutting once the average quality within the window falls below a threshold.
* **_LEADING_** Cut bases off the start of a read, if below a threshold quality.
* **_TRAILING_** Cut bases off the end of a read, if below a threshold quality.
* **_CROP_** Cut the read to a specified length.
* **_HEADCROP_** Cut the specified number of bases from the start of the read.
* **_MINLEN_** Drop an entire read if it is below a specified length.
* **_TOPHRED33_** Convert quality scores to Phred-33.
* **_TOPHRED64_** Convert quality scores to Phred-64.


A general command for *Trimmomatic* on this cluster will look something like this:

```
$ java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar SE \
-threads 4 \
inputfile \
outputfile \
OPTION:VALUE... # DO NOT RUN THIS
```
`java -jar` calls the Java program, which is needed to run *Trimmomatic*, which is a 'jar' file (`trimmomatic-0.33.jar`). A 'jar' file is a special kind of java archive that is often used for programs written in the Java programming language.  If you see a new program that ends in '.jar', you will know it is a java program that is executed `java -jar` <*location of program .jar file*>.  The `SE` argument is a keyword that specifies we are working with single-end reads. We have to specify the `-threads` parameter because Trimmomatic uses 16 threads by default.

The next two arguments are input file and output file names. These are then followed by a series of options. The specifics of how options are passed to a program are different depending on the program. You will always have to read the manual of a new program to learn which way it expects its command-line arguments to be composed.

###Running Trimmomatic

Change directories to the untrimmed fastq data location:

`$ cd ~/unix_oct2015/rnaseq_project/data/untrimmed_fastq`

Let's load the trimmomatic module:

`$ module load seq/Trimmomatic/0.33`

Since the *Timmomatic* command is complicated and we will be running it a number of times, let's draft the command in a text editor, such as Sublime, TextWrangler or Notepad++. When finished, we will copy and paste the command into the terminal.

For the single fastq input file 'Mov10_oe_1.subset.fq', the command would be:

```
$ java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar SE \
-threads 4 \
-phred33 \
Mov10_oe_1.subset.fq \
../trimmed_fastq/Mov10_oe_1.qualtrim25.minlen35.fq \
ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 \
TRAILING:25 \
MINLEN:35
```
The backslashes at the end of the lines allow us to continue our script on new lines, which helps with readability of some long commands.

This command tells *Trimmomatic* to run on a fastq file containing Single-End reads (``Mov10_oe_1.subset.fq``, in this case) and to name the output file ``Mov10_oe_1.qualtrim25.minlen35.fq``. The program will remove Illumina adapter sequences given by the file, `TruSeq3-SE.fa` and will cut nucleotides from the 3' end of the sequence if their quality score is below 25. The entire read will be discarded if the length of the read after trimming drops below 35 nucleotides.

After the job finishes, you should see the *Trimmomatic* output in the terminal: 

```
TrimmomaticSE: Started with arguments: -threads 4 -phred33 Mov10_oe_1.subset.fq ../trimmed_fastq/Mov10_oe_1.qualtrim25.minlen35.fq ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 TRAILING:25 MINLEN:35
Using Long Clipping Sequence: 'AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGTA'
Using Long Clipping Sequence: 'AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC'
ILLUMINACLIP: Using 0 prefix pairs, 2 forward/reverse sequences, 0 forward only sequences, 0 reverse only sequences
Input Reads: 305900 Surviving: 300423 (98.21%) Dropped: 5477 (1.79%)
TrimmomaticSE: Completed successfully
```

Now that we know the command successfully runs, let's make the *Trimmomatic* command into a submission script. A submission script is oftentimes preferable to executing commands on the terminal. We can use it to store the parameters we used for a command(s) inside a file. If we need to run the program on other files, we can easily change the script. Also, using scripts to store your commands helps with reproducibility. In the future, if we forget which parameters we used during our analysis, we can just check our script.

#### Running our *Trimmomatic* command as a job submission to the LSF scheduler

To run the *Trimmomatic* command on a worker node via the job scheduler, we need to create a submission script with two important components:

1. our **LSF directives** at the **beginning** of the script. This is so that the scheduler knows what resources we need in order to run our job on the compute node(s).

2. the commands to be run in order


To make a *Trimmomatic* job submission script for Orchestra LSF scheduler:

`$ cd ~/unix_oct2015`

`$ vi trimmomatic_mov10.lsf`

Within nano we will add our shebang line, the Orchestra job submission commands, and our Trimmomatic command. Remember that you can find the submission commands in the [Orchestra New User Guide](https://wiki.med.harvard.edu/Orchestra/NewUserGuide).


```
#!/bin/bash

#BSUB -q priority # queue name
#BSUB -W 2:00 # hours:minutes runlimit after which job will be killed.
#BSUB -n 4 # number of cores requested
#BSUB -J rnaseq_mov10_trim         # Job name
#BSUB -o %J.out       # File to which standard out will be written
#BSUB -e %J.err       # File to which standard err will be written

cd ~/unix_oct2015/rnaseq_project/data/untrimmed_fastq

java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar SE \
-threads 4 \
-phred33 \
Mov10_oe_1.subset.fq \
../trimmed_fastq/Mov10_oe_1.qualtrim25.minlen35.fq \
ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 \
TRAILING:25 \
MINLEN:35
```
Now, let's run it:

`$ bsub < trimmomatic_mov10.sh`

After the job finishes, you should receive an email with output: 

```
TrimmomaticSE: Started with arguments: -threads 4 -phred33 Mov10_oe_1.subset.fq ../trimmed_fastq/Mov10_oe_1.qualtrim25.minlen35.fq ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 TRAILING:25 MINLEN:35
Using Long Clipping Sequence: 'AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGTA'
Using Long Clipping Sequence: 'AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC'
ILLUMINACLIP: Using 0 prefix pairs, 2 forward/reverse sequences, 0 forward only sequences, 0 reverse only sequences
Input Reads: 305900 Surviving: 300423 (98.21%) Dropped: 5477 (1.79%)
TrimmomaticSE: Completed successfully
```

We now have a new fastq file with our trimmed and cleaned up data:

`$ ls ../trimmed_fastq/`    


***
**Exercise**

**Run *Trimmomatic* on all the fastq files**

Now we know how to run *Trimmomatic*, but there is some good news and bad news.  
One should always ask for the bad news first.  *Trimmomatic* only operates on 
one input file at a time and we have more than one input file.  The good news?
We already know how to use a 'for loop' to deal with this situation. Let's modify our script to run the *Trimmomatic* command for every raw fastq file. Let's also run *FastQC* on each of our trimmed fastq files to evaluate the quality of our reads post-trimming:

```
#!/bin/bash

#BSUB -q priority 		# queue name
#BSUB -W 2:00 		# hours:minutes runlimit after which job will be killed.
#BSUB -n 6 		# number of cores requested
#BSUB -J rnaseq_mov10_qc         # Job name
#BSUB -o %J.out       # File to which standard out will be written
#BSUB -e %J.err       # File to which standard err will be written

cd ~/unix_oct2015/rnaseq_project/data/untrimmed_fastq

module load seq/Trimmomatic/0.33
module load seq/fastqc/0.11.3

for infile in *.fq; do

  outfile=$infile.qualtrim25.minlen35.fq;

  java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar SE \
  -threads 4 \
  -phred33 \
  $infile ../trimmed_fastq/$outfile \
  ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 \
  TRAILING:25 \
  MINLEN:35;
  
done
    
fastqc -t 6 ../trimmed_fastq/*.fq
```
`$ bsub < trimmomatic_mov10.sh`

It is good practice to load the modules we plan to use at the beginning of the script. Therefore, if we run this script in the future, we don't have to worry about whether we have loaded all of the necessary modules prior to executing the script. 

Do you remember how the variable name in the first line of a 'for loop' specifies a variable that is assigned the value of each item in the list in turn?  We can call it whatever we like.  This time it is called `infile`.  Note that the fifth line of this 'for loop' is creating a second variable called `outfile`.  We assign it the value of `$infile` with `'.qualtrim25.minlen35.fq'` appended to it. **There are no spaces before or after the '='.**

After we have created the trimmed fastq files, we wanted to make sure that the quality of our reads look good, so we ran a *FASTQC* on our `$outfile`, which is located in the ../trimmed_fastq directory.

Let's make a new directory for our fasqc files for the trimmed reads:

`$ mkdir ~/unix_oct2015/rnaseq_project/results/fastqc_trimmed_reads`

Now move all fastqc files to the `fastqc_trimmed_reads` directory:

`$ mv ~/unix_oct2015/rnaseq_project/data/trimmed_fastq/*fastqc**`

Let's use *FileZilla* to download the fastqc html for Mov10_oe_1. Has our read quality improved with trimming?

#### Running our *Trimmomatic* command with paired-end data


![paired-end_data](../img/paired-end_data.png)

**
How would our command change if we were using paired-end data?**

```
java -jar ~/tools/Trimmomatic-0.33/trimmomatic-0.33.jar PE \
-phred33 \
-threads 4 \
-trimlog raw_fastq/trimmomatic.log \
<input 1> <input 2> \
<paired output 1> <unpaired output 1> \
<paired output 2> <unpaired output 2> \
ILLUMINACLIP:/home/rsk27/tools/Trimmomatic-0.33/adapters/TruSeq3-PE.fa:2:30:10 \
TRAILING:25 \
MINLEN:35
```
***
*The materials used in this lesson was derived from work that is Copyright © Data Carpentry (http://datacarpentry.org/). 
All Data Carpentry instructional material is made available under the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0).*