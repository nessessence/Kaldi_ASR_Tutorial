# Kaldi: auto speech recognition tutorial 
This repositoty is mainly modified from this [yesno_tutorial](https://github.com/ekapolc/ASR_classproject/tree/master/yesnotutorial). and all the references are addressed below the tutorial.

This tutorial will guide you through some basic functionalities and operations of [Kaldi](http://kaldi-asr.org/) ASR toolkit which can be applied in any general auto speech recognition tasks.
In this tutorial, we will use [VoxForge](http://www.voxforge.org/home/downloads) dataset which is one of the most popular datasets for auto speech recognition.

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

each directory in "VF_Main_16kHz" has a unique speakerID and contains two directories
  - wav : contains `.wav` files
  - etc : contains the information files of each `.wav` file.(In this tutorial, we'll focus on `prompts-original` file which contains the string of the name of `.wav` file and its transcription for each line)

This is all we have as our raw data. Now we will deform these `.wav` files into data format that Kaldi can read in.

### Data preparation
Let's start with formatting data. We will randomly split wave files into test and train dataset(set the ratio as you want). Create a directory data and,then two subdirectories train and test in it.

Now, for each dataset (train, test), we need to generate these files representing our raw data - the audio and the transcripts.

* `text`
    * Essentially, transcripts.
    * An utterance per line, `<utt_id> <transcript>` 
        * e.g. `Aaron-20080318-kdl_b0019 HIS SLIM HANDS GRIPPED THE EDGES OF THE TABLE `
    * We will use filenames without extensions as utt_ids for now.
    * Although recordings are in Hebrew, we will use English words, YES and NO, to avoid complicating the problem.
* `wav.scp`
    * Indexing files to unique ids. 
    * `<file_id> <wave filename with path OR command to get wave file>`
        * e.g. `Aaron-20080318-kdl_b0019 /mnt/data/VF_Main_16kHz/Aaron-20080318-kdl/wav/b0019.wav
`
    * Again, we can use file names as file_ids.
* `utt2spk`
    * For each utterance, mark which speaker spoke it.
    * `<utt_id> <speaker_id>`
        * e.g. `Aaron-20080318-kdl_b0019 Aaron`
    * Since we have only one speaker in this example, let's use "global" as speaker_id
* `spk2utt`
    * Simply inverse indexed `utt2spk` (`<speaker_id> <all_hier_utterences>`)
* `full_vocab` : list of the vocabulary of in text of training data. *only for train directory
* (optional) `segments`: *not used for this data.*
    * Contains utterance segmentation/alignment information for each recording. 
    * Only required when a file contains multiple utterances, which is not this case.
* (optional) `reco2file_and_channel`: *not used for this data. *
    * Only required when audios were recorded in dual channels for conversational setup.
* (optional) `spk2gender`: not used for this data. 
    * Map from speakers to their gender information. 
    * Used in vocal tract length normalization. 
    
Our task is to generate these files. You can use this python notebook [preparation_data.ipynb](https://github.com/nessessence/Kaldi_VoxForge/blob/master/data_preparation.ipynb). but if this's your first time in Kaldi, I encourage you to write your own script because it'll improve your understanding of Kaldi format.
Note: you can generate the "spk2utt" file using Kaldi utility: 
```utils/utt2spk_to_spk2utt.pl data/train/utt2spk > data/train/spk2utt```

#####
```bash
utils/fix_data_dir.sh data/train/
utils/fix_data_dir.sh data/test/
```

If you're done with the code, your data directory should look like this, at this point. 
```
data
├───train
│   ├───text
│   ├───utt2spk
│   ├───spk2utt
│   |───wav.scp
|   └───full_vocab           //only for train directory
└───test
    ├───text
    ├───utt2spk
    ├───spk2utt
    └───wav.scp
```
## Step 2 - Dictionary preparation

This section will cover how to build language knowledge - lexicon and phone dictionaries - for Kaldi recognizer.

### Before moving on

From here, we will use several Kaldi utilities (included in `steps` and `utils` directories) to process further. To do that, Kaldi binaries should be in your `$PATH`. 
However, Kaldi is a huge framework, and there are so many binaries distributed over many different directories, depending on their purpose. 
So, we will use a script, `path.sh` to add all of them to `$PATH` of the subshell every time a script runs (we will see this later).
All you need to do right now is to open the `path.sh` file and edit the `$KALDI_ROOT` variable to point your Kaldi Installation location. 

### Defining blocks of the language: Lexicon

Next we will build dictionaries. Let's start with creating intermediate `dict` directory at the root.

```bash
vf# mkdir -p data/local/dict
```
Your `dict` directory should contain these 5 files:

* `lexicon.txt`: list of word-phone pairs
* `lexiconp.txt` : list of word-prob-phone pairs
* `silence_phones.txt`: list of silent phones
* `nonsilence_phones.txt`: list of non-silent phones
* `optional_silence.txt`: list of optional silent phones (here, this looks the same as `silence_phones.txt`)
  
Let's look at each file format and example.

lexicon.txt: ``` <word> <phone1> <phone2>  …. <phoneN> ```  in an alphabetical order of word.

```bash
vf# head -5 data/local/dict/lexicon.txt
A	AH
A	EY
ABANDONMENT	AH B AE N D AH N M AH N T
ABLE	EY B AH L
ABNORMAL	AE B N AO R M AH L
```
in order to generate lexicon.txt file, you can use ``` /local/utils/prepare_dict.sh  ```
which will create:

Note: as you can see, lexicon.txt will contain repeated entries for the same word on separate lines, if we have multiple pronunciations for it. 


`lexiconp.txt`: ``` <word> <pron-prob> <phone1> <phone2>  …. <phoneN> ```  #similar to lexicon.txt, just add the pronounciation-probability term.




However, in real speech, there are not only human sounds that contributes to a linguistic expression, but also silence and noises. 
Kaldi calls all those non-linguistic sounds "*silence*".
For example, even in this small, controlled recordings, we have pauses between each word. 
Thus we need an additional phone "SIL" representing silence. And it can be happening at end of of all words. Kaldi calls this kind of silence "*optional*".

```bash
echo "SIL" > data/local/dict/silence_phones.txt
echo "SIL" > data/local/dict/optional_silence.txt
mv data/local/dict/phones.txt data/local/dict/nonsilence_phones.txt
```

Now amend lexicon to include the silence as well.

```bash
cp data/local/dict/lexicon.txt data/local/dict/lexicon_words.txt
echo "<SIL> SIL" >> data/local/dict/lexicon.txt 
```
**Note** that "\<SIL\>" will also be used as our OOV token later.



Finally, we need to convert our dictionaries into a data structure that Kaldi would accept - finite state transducer (FST). Among many scripts Kaldi provides, we will use `utils/prepare_lang.sh` to generate FST-ready data formats to represent our language definition.

```bash
utils/prepare_lang.sh --position-dependent-phones false <RAW_DICT_PATH> <OOV> <TEMP_DIR> <OUTPUT_DIR>
```
We're using `--position-dependent-phones` flag to be false in our tiny, tiny toy language. There's not enough context, anyways. For required parameters we will use: 

* `<RAW_DICT_PATH>`: `data/local/dict`
* `<OOV>`: `"<SIL>"`
* `<TEMP_DIR>`: Could be anywhere. I'll just put a new directory `tmp` inside `dict`.
* `<OUTPUT_DIR>`: This output will be used in further training. Set it to `data/lang`.


### Defining sequence of the blocks: Language model

We are given a sample uni-gram language model for the yesno data. 
You'll find a `arpa` formatted language model inside `data/local` directory. 
However, again, the language model also needs to be converted into a FST.
For that, Kaldi also comes with a number of programs.
In this example, we will use the script, `local/prepare_lm.sh`.
It will generate properly formatted LM FST and put it in `data/lang_test_tg`.


### references and useful resources
[official Kaldi document](https://kaldi-asr.org/doc)


