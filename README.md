# Kaldi: VoxForge tutorial 
This repositoty is modified from [yesno_tutorial](https://github.com/ekapolc/ASR_classproject/tree/master/yesnotutorial)

This tutorial will guide you through some basic functionalities and operations of [Kaldi](http://kaldi-asr.org/) ASR toolkit using [VoxForge](http://www.voxforge.org/home/downloads) dataset which is one of the most popular datasets for auto speech recognition.

## Step 0 - Installing Kaldi  

## Requirements

The Kaldi will run on POSIX systems, with these software/libraries pre-installed.
(If you don't know how to use a package manager on your computer to install these libraries, this tutorial might not be for you.)

* [GNU build tools](https://en.wikipedia.org/wiki/GNU_Build_System#Components)
* [`wget`](https://www.gnu.org/software/wget/)
* [`git`](https://git-scm.com/)
* (optional) [`sox`](http://sox.sourceforge.net/)

Recommendation: For Windows users, although Kaldi is supported in Windows, I highly recommend you to install it in a container of the UNIX operating system such as  Linux.

The entire compilation can take a couple of hours and up to 8 GB of storage depending on your system specification and configuration. Make sure you have enough resource before start compiling.

## Compilation 

Once you have all required build tools, compiling the Kaldi is pretty straightforward. First you need to download it from the repository.

```bash
git clone https://github.com/kaldi-asr/kaldi.git /path/you/want --depth 1
cd /path/you/want
```
(`--depth 1`: You might want to give this option to shrink the entire history of the project into a single commit to save your storage and bandwidth.)

Assuming you are in the directory where you cloned (downloaded) Kaldi, now you need to perform `make` in two subdirectories: `tools`, and `src`

```bash
cd tools/
make
cd ../src
./configure
make depend
make
```
If you need more detailed install instructions or having trouble/errors while compiling, please check out the official documentation: [tools/INSTALL](https://github.com/kaldi-asr/kaldi/blob/master/tools/INSTALL), [src/INSTALL](https://github.com/kaldi-asr/kaldi/blob/master/src/INSTALL)

Now all the Kaldi tools should be ready to use.

## Step 1 - Data preparation

This section will cover how to prepare your data to train and test a Kaldi recognizer.

### Download dataset

there are two options to download VoxForge dataset
  - directly download from [VoxForge website](http://www.repository.voxforge1.org/downloads/SpeechCorpus/Trunk/Audio/Main/16kHz_16bit)
  - donwload using Kaldi script ```/kaldi/egs/voxforge/s5/getdata.sh```


### Data description

VoxForge dataset has 95628 `.wav` files, sampled at 16 kHz by 1235 identified speakers and 2164 anonymous speakers(for this tutorial, speech recognition task is speaker independent which doesn't care about the speakers)

each folder in "VF_Main_16kHz" has a unique speakerID and contains two subfolders
  - wav : contains `.wav` files
  - etc : contains the information files of each `.wav` file.(In this tutorial, we'll focus on `prompts-original` file which contains the string of the name of `.wav` file and its transcription for each line)

This is all we have as our raw data. Now we will deform these `.wav` files into data format that Kaldi can read in.

### Data preparation
Let's start with formatting data. We will randomly split wave files into test and train dataset. Create a directory data and,then two subdirectories train and test in it.

Now, for each dataset (train, test), we need to generate these files representing our raw data - the audio and the transcripts.

* `text`
    * Essentially, transcripts.
    * An utterance per line, `<utt_id> <transcript>` 
        * e.g. `0_0_1_1_1_1_0_0 NO NO YES YES YES YES NO NO`
    * We will use filenames without extensions as utt_ids for now.
    * Although recordings are in Hebrew, we will use English words, YES and NO, to avoid complicating the problem.
* `wav.scp`
    * Indexing files to unique ids. 
    * `<file_id> <wave filename with path OR command to get wave file>`
        * e.g. `0_1_0_0_1_0_1_1 waves_yesno/0_1_0_0_1_0_1_1.wav`
    * Again, we can use file names as file_ids.
* `utt2spk`
    * For each utterance, mark which speaker spoke it.
    * `<utt_id> <speaker_id>`
        * e.g. `0_0_1_0_1_0_1_1 global`
    * Since we have only one speaker in this example, let's use "global" as speaker_id
* `spk2utt`
    * Simply inverse indexed `utt2spk` (`<speaker_id> <all_hier_utterences>`)
    * Can use a Kaldi utility to generate
    * `utils/utt2spk_to_spk2utt.pl data/train_yesno/utt2spk > data/train_yesno/spk2utt`
* (optional) `segments`: *not used for this data.*
    * Contains utterance segmentation/alignment information for each recording. 
    * Only required when a file contains multiple utterances, which is not this case.
* (optional) `reco2file_and_channel`: *not used for this data. *
    * Only required when audios were recorded in dual channels for conversational setup.
* (optional) `spk2gender`: not used for this data. 
    * Map from speakers to their gender information. 
    * Used in vocal tract length normalization. 
our task is to generate this file. you can use
I encourage

 



