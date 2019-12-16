# Kaldi: auto speech recognition tutorial 
This repository is mainly modified from this [yesno_tutorial](https://github.com/ekapolc/ASR_classproject/tree/master/yesnotutorial). and all the references are addressed below the tutorial.

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

Recommendation: For Windows users, although Kaldi is supported in Windows, I highly recommend you to install Kaldi in a container of the UNIX operating system such as Linux.

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
* `full_vocab` : list of all the vocabulary in the text of training data. (this file will be used for making the dictionary)   
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
|   └───full_vocab  *only for train directory
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
Your `dict` directory should contain at least these 5 files:

* `lexicon.txt`: list of word-phone pairs
* (optional)`lexiconp.txt` : list of word-prob-phone pairs
* `silence_phones.txt`: list of silent phones 
* `nonsilence_phones.txt`: list of non-silent phones (including various kinds of noise, laugh, cough, filled pauses etc)
* `optional_silence.txt`: contains just a single phone (typically SIL)
  
we can use `/utils/prepare_dict.sh` to generate all the files above excluding `lexiconp.txt`
brief explaination for the command `/utils/prepare_dict.sh`: 
1. downloads the general word-phone pairs open source dictionary ( this tutorial uses "cmudict" ).
2. the pairs of the word which contained in both the general dictionary and `full_vocab` will be in `lexicon-iv.txt`.
3. the words which contained in the `full_vocab`(which we have been generated since data preparation), but not in the general dictionary will be contained in `vocab-oov.txt` (oov standfor "out-of-vocab").
4. generates the pronounciations of those oov-vocab using a pre-trained Sequitur G2P model in `conf/g2p_model` and stores the pairs in `lexicon-oov.txt`.
5. merges `lexicon-iv.txt` and `lexicon-oov.txt` then adds the silence symbol (typically (<SIL>,SIL)) at the end to generate the `lexicon.txt`.
6. generates the other files. 

Note: all the files are in an alphabetical order. and you change the parameter `ss` at the top in ` /utils/prepare_dict.sh ` file to set the silence symbol as you want. (In this tutorial use `<SIL>` as the silence symbol )

Let's look at each file format and overview.

lexicon.txt: ``` <word> <phone1> <phone2>  .....  <phoneN> ```  in an alphabetical order of word.

```bash
vf# head -5 data/local/dict/lexicon.txt
A	AH
A	EY
ABANDONMENT	AH B AE N D AH N M AH N T
ABLE	EY B AH L
ABNORMAL	AE B N AO R M AH L
```
Note: as you can see, lexicon.txt will contain repeated entries for the same word on separate lines, if we have multiple pronunciations for it. 

`lexiconp.txt`: ``` <word> <pron-prob> <phone1> <phone2>  …. <phoneN> ```  #similar to lexicon.txt, just add the pronounciation-probability term.

`silence_phones.txt`:
```bash
vf# more data/local/dict/silence_phones.txt
SIL
```
`nonsilence_phones.txt`:
```bash
vf# head -10 data/local/dict/nonsilence_phones.txt
AA
AE
AH
AO
AW
AY
B
CH
D
DH
```
`optional_silence.txt`:
```bash
vf# more data/local/dict/optional_silence.txt 
SIL
```
**Note** that `<SIL>` will also be used as our OOV token later.

Finally, we need to convert our dictionaries into a data structure that Kaldi would accept - finite state transducer (FST). Among many scripts Kaldi provides, we will use `utils/prepare_lang.sh` to generate FST-ready data formats to represent our language definition.

```bash
vf# utils/prepare_lang.sh --position-dependent-phones false <RAW_DICT_PATH> <OOV> <TEMP_DIR> <OUTPUT_DIR>
```
We're using `--position-dependent-phones` flag to be false in our tiny, tiny toy language. There's not enough context, anyways. For required parameters we will use: 

* `<RAW_DICT_PATH>`: `data/local/dict`
* `<OOV>`: `"<SIL>"`
* `<TEMP_DIR>`: Could be anywhere. I'll just put a new directory `tmp` inside `dict`.
* `<OUTPUT_DIR>`: This output will be used in further training. Set it to `data/lang`.

```bash
vf# ls data/lang
L.fst  L_disambig.fst  oov.int	oov.txt  phones  phones.txt  topo  words.txt
```

## Step 3 - Feature extraction and training

This section will cover how to perform MFCC feature extraction and GMM modeling.

### Feature extraction

Once we have all data ready, it's time to extract features for GMM training.

First extract mel-frequency cepstral coefficients.

```bash
vf# steps/make_mfcc.sh --nj <N> <INPUT_DIR> <LOG_DIR> <OUTPUT_DIR> 
```

* `--nj <N>` : number of processors, defaults to 4. Kaldi splits the processes by speaker information. Therefore, `nj` must be lesser than or equal to the number of speakers in `<INPUT_DIR>`. For this simple tutorial which has 1 speaker, `nj` must be 1.
* `<INPUT_DIR>` : where we put our 'data' of training set
* `<LOG_DIR>` : directory to dumb log files. Let's put output to `exp/make_mfcc/train_yesno`, following Kaldi recipes convention
* `<OUTPUT_DIR>` : Directory to put the features. The convention uses `mfcc/train`

```bash
vf# ls mfcc/train
raw_mfcc_train.1.ark  raw_mfcc_train.2.scp  raw_mfcc_train.4.ark
raw_mfcc_train.1.scp  raw_mfcc_train.3.ark  raw_mfcc_train.4.scp
raw_mfcc_train.2.ark  raw_mfcc_train.3.scp
```
Now normalize cepstral features using Cepstral Mean Normalization just like we did in our previous homework. This step also does an extra variance normalization. Thus, the process is called Cepstral Mean and Variance Normalization (CMVN).


```bash
vf# steps/compute_cmvn_stats.sh <INPUT_DIR> <LOG_DIR> <OUTPUT_DIR>
```
`<INPUT_DIR>`, `<LOG_DIR>`, and `<OUTPUT_DIR>` are the same as above.

The two scripts will create `wav.scp` and `cmvn.scp` which specifies where the computed MFCC and CMVN are. `wav.scp` and `cmvn.scp` are just text files with just `<utt_id> <path_to_data>` for each line. With this setup, by passing the `data/train` directory to a Kaldi script, you are passing various information, such as the transcription, the location of the wav file, or the MFCC features.

**Note** that these shell scripts (`.sh`) are all pipelines through Kaldi binaries with trivial text processing on the fly. To see which commands were actually executed, see log files in `<LOG_DIR>`. Or even better, see inside the scripts. For details on specific Kaldi commands, refer to [the official documentation](http://kaldi-asr.org/doc/tools.html).

### Training Acoustic Models   
In this step, we'll train acoustic model using Kaldi Utilities.
you can follow this `train.sh` 
for example: monophone model training
```bash 
vf# steps/train_mono.sh --nj <N> --cmd <MAIN_CMD> --totgauss 400 <DATA_DIR> <LANG_DIR> <OUTPUT_DIR>
```
* `--cmd <MAIN_CMD>`: To use local machine resources, use `"utils/run.pl"` pipeline.
* `--totgauss : limits the number of gaussian mixtures to 400
* `--nj <N>`: Utterances from a speaker cannot be processed in parallel. Since we have only one, we must use 1 job only. 
* `<DATA_DIR>`: Path to our training 'data'
* `<LANG_DIR>`: Path to language definition (output of the `prepare_lang` script)
* `<OUTPUT_DIR>`: like the previous, use `exp/mono`.

When you run the command, you will notice it doing EM. Each iteration does an alignment stage and an update stage. 
This will generate FST-based lattice for acoustic model. Kaldi provides a tool to see inside the model (which may not make any sense now).

```bash
/path/to/kaldi/src/fstbin/fstcopy 'ark:gunzip -c exp/mono/fsts.1.gz|' ark,t:- | head -n 20
```
This will print out first 20 lines of the lattice in human-readable(!!) format (Each column indicates: Q-from, Q-to, S-in, S-out, Cost)

Note: the training in `train.sh` is important. for example, inorder to `train tri1`, you have to train monophone and then alignment first.
you don't have to follow all the training sequence in `train.sh`. it depends on complexity of your data.
eg. for yes-no dataset, maybe just a monophone training is enough.

## Step 4 - Decoding and testing

This section will cover decoding of the model we trained.

### Graph decoding

Now we're done with acoustic model training. 
For decoding, we need a new input that goes over our lattices of AM & LM. 
In step 1, we prepared separate testset in `data/test_yesno` for this purpose. 
Now it's time to project it into the feature space as well.
Use `steps/make_mfcc.sh` and `steps/compute_cmvn_stats.sh` .

Then, we need to build a fully connected FST (HCLG) network. 

```bash
vf# utils/mkgraph.sh --mono data/lang_test_tg exp/mono exp/mono/graph_tgpr
```
This will build a connected HCLG in `exp/mono/graph_tgpr` directory. 

Finally, we need to find the best paths for utterances in the test set, using decode script. Look inside the decode script, figure out what to give as its parameter, and run it. Write the decoding results in `exp/mono/decode_test_yesno`.

```bash 
vf# steps/decode.sh 
```

This will end up with `lat.N.gz` files in the output directory, where N goes from 1 up to the number of jobs you used (which must be 1 for this task). These files contain lattices from utterances that were processed by N’th thread of your decoding operation.


### Looking at results

If you look inside the decoding script, it ends with calling the scoring script (`local/score.sh`), which generates hypotheses and computes word error rate of the testset 
See `exp/mono/decode_test_yesno/wer_X` files to look the WER's, and `exp/mono/decode_test_yesno/scoring/X.tra` files for transcripts. 
`X` here indicates language model weight, *LMWT*, that scoring script used at each iteration to interpret the best paths for utterances in `lat.N.gz` files into word sequences. (Remember `N` is #thread during decoing operartion)
You can deliberately specify the weight using `--min_lmwt` and `--max_lmwt` options when `score.sh` is called, if you want. 
(See lecture slides on decoding to refresh what LMWT is, if you are not sure)

Or if you are interested in getting word-level alignment information for each reocoding file, take a look at `steps/get_ctm.sh` script.


### references and useful resources
[official Kaldi document](https://kaldi-asr.org/doc)
https://github.com/keighrim/kaldi-yesno-tutorial/blob/master/README.md


