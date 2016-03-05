---
title: "Peak calling with SPP"
author: "Meeta Mistry"
date: "Thursday, March 3rd, 2016"
---

Contributors: Meeta Mistry

Approximate time: 90 minutes

## Learning Objectives

* Learning how to use SPP for peak calling
* Understanding different components of the SPP algorithm
* Interpretation of results from a peaks including cross-correlation plots and narrowPeak files 

## Peak calling

Peak calling, the next step in our workflow, is a computational method used to identify areas in the genome that have been enriched with aligned reads as a consequence of performing a ChIP-sequencing experiment. 

<div style="text-align:center"><img src="../img/workflow-peakcalling.png" width="400"></div>


From the alignment files (BAM), you typically observe reads/tags to be identified on each side of the binding site for the protein of interest. The 5' ends of the selected fragments will form groups on the positive- and negative-strand. The distributions of these groups are then assessed using statistical measures and compared against background (input or mock IP samples) to determine if the binding site is significant.


<div style="text-align:center"><img src="../img/chip-fragments.png" width="200" align="middle"></div>

There are various tools that are available for peak calling. Two of the popular one we will demonstrate in this session are SPP and MACS2. *Note that for this lesson the term tag and sequence read are interchangeable.*

## SPP

SPP is a data processing pipeline optimized for detection of localized protein binding positions from unpaired sequence reads. The [publication](http://www.nature.com.ezp-prod1.hul.harvard.edu/nbt/journal/v26/n12/full/nbt.1508.html) describes the algorithm in great detail, from which we have listed some of the main features here.

* Discarding or restricting positions with abnormally high number of tags
* Provides smoothed tag density in WIG files for viewing in other browsers
* Provides conservative statistical estimates of fold enrichment ratios along the genome, to determine regions of significant enrichment/depletion (can be exported for visualization)

The main steps of the ChIP-seq processing pipline are described in the illustration below. As we walk through the SPP pipeline in this lesson, we will describe each step in more detail.

<div style="text-align:center"><img src="../img/spp-fig1.png"></div>


SPP is an R package which can be installed in one of two ways. There is [source code](https://github.com/hms-dbmi/spp/archive/1.13.tar.gz) avaiable for download, or alternatively it can be installed using `devtools` as it is now [available on GitHub](https://github.com/hms-dbmi/spp).



## Setting up

Exit your current session on Orchestra if you are currently in an interactive session, and restart one using default settings. Since we are working with such a small dataset we will just use a single core, for parallel processing options with SPP see note below.

	$ bsub -Is -q interactive bash

Now let's setup the directory structure. Navigate to `~/ngs_course/chipseq/` if you are not already there. Within the results directory we will create directory called `spp`:

	$ mkdir results/spp
	
The last thing we need to before getting started is load the appropriate	software. As mentioned, SPP is an R package. On Orchestra it comes installed by default when you load the most recent R module:

	$ module load stats/R/3.2.1

> ### Parallel processing with SPP
> 	
> When working with large datasets it can be beneficial to use multiple cores during some of the more computationally intensive processes. In order to do so, you will need to install the `snow` packagein R. Using snow you can initialize a cluster of nodes for parallel processing (in the example below we have a cluster of 8 nodes). *See `snow` package manual for details.* This cluster variable can then be used as input to functions that allow for parallel processing.
> 
> 	`library(snow)`
> 	
> 	`cluster <- makeCluster(8)`


## An R script for running SPP

To run SPP, there are several functions that need to be run sequentially. For more information on these functions the [home page](http://compbio.med.harvard.edu/Supplements/ChIP-seq/) is quite useful, as they provide a brief tutorial showing the use of the main methods.

For this class, we have put together an R script that contains all of the methods required for peak calling. You can copy over the script into your current directory, and then we can discuss the methods in more detail.

	$ cp /groups/hbctraining/ngs-data-analysis2016/chipseq/scripts/get_peaks.R .

Open it up using `vim`, as there is a modification we need to make in order for you to be able to run this from your working directory. Use `:set number` in `vim` to add numbers to your lines. Now scroll down to line 16. Here, you need to change the path to where your `spp` directory is located. It will look something like this:

	/home/user_name/ngs_course/chipseq/results/spp

Save and exit vim to avoid making any other changes. You can open up the script again using `less` as we describe the code or just read it in the markdown.

### Setup the environment

The first few lines are setting up the environment which involves **loading the library and reading in the data**. The input and treatment BAM files need to be given as arguments to this script when running it. The final few lines in this chunk of code include defining a path for the resulting output files and a prefix for output file names.

```
# Load library
library(spp)

# Get filenames from arguments
filenames <- commandArgs(trailingOnly=TRUE)
file.input <- filenames[1]
file.data <- filenames[2]

# Load in data
input.data <- read.bam.tags(file.input, read.tag.names=T)
chip.data <- read.bam.tags(file.data, read.tag.names=T)

# Set path 
path <- "/groups/hbctraining/ngs-data-analysis2016/chipseq/spp/"

# Create a prefix for your output file
# This can be changed based on file naming convention 
s <- strsplit(file.data,split="_")
prefix <- paste(s[[1]][2], "_", s[[1]][3], sep="")
``` 

### Remove anomalous features

The next chunk of code **uses the cross-correlation profile to calculate binding peak separation distance**.  The separation distance will be printed out and the **cross-correlation plot** will be saved to file. The `srange` argument gives the possible range for the size of the protected region; it should be higher than tag length but note that making the upper boundary too high will increase calculation time. The `bin` argument is telling SPP to bin tags within the specified number of basepairs to speed up calculation. Increasing the bin size decreases the accuracy of the determined parameters. The numbers we have selected here are defaults suggested in the tutorial.

At this point SPP also assesses whether the inclusion of **reads with non-perfect alignment quality** improves the cross-correlation peak, and flags them accordingly. If you would like to accept all aligned tags, specify `accept.all.tags=T` argument to save time.


```
# Get binding info from cross-correlation profile
binding.characteristics <- get.binding.characteristics(chip.data,srange=c(50,500),bin=5)

# Print out binding peak separation distance
print(paste("binding peak separation distance =",binding.characteristics$peak$x))

# Plot cross-correlation profile
pdf(file=paste(path, prefix, ".crosscorrelation.pdf", sep=""),width=5,height=5)
par(mar = c(3.5,3.5,1.0,0.5), mgp = c(2,0.65,0), cex = 0.8)
plot(binding.characteristics$cross.correlation,type='l',xlab="strand shift",ylab="cross-correlation")
abline(v=binding.characteristics$peak$x,lty=2,col=2)
dev.off()
```

### Assemble informative tags

The next function will select tags with acceptable alignment quality, based on flags assigned above. Moving forward with only informative tags, the chip and input data is now a simple list of tag coordinate vectors (read start position:read end position). 

The next function will scan along the chromosomes calculating local density of regions (can be specified using window.size parameter, default is 200bp), removing or restricting singular positions with extremely high tag count relative to the neighborhood.

<div style="text-align:center"><img src="../img/read-density.png"></div>

```
# select informative tags based on the binding characteristics
chip.data <- select.informative.tags(chip.data, binding.characteristics)
input.data <- select.informative.tags(input.data, binding.characteristics)

# restrict or remove singular positions with very high tag counts
chip.data <- remove.local.tag.anomalies(chip.data)
input.data <- remove.local.tag.anomalies(input.data)
```


