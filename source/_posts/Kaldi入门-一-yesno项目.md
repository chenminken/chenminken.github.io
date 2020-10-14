---
title: 'Kaldi入门(一):yesno项目'
date: 2020-05-01 17:37:05
tags: [Kaldi, ASR]
---

这个学期选了一门自然语言处理课，结果这门课主要的研究课题是自动语音识别（ASR）。既然入了这个坑。就先好好了解一下如何做ASR吧。
老师Tom Ko要求使用Kaldi这个工具来做ASR。课上到一半才知道Kaldi中有几千行的脚本代码是老师提交的。好吧，脚本好难的。
为了入门Kaldi，课程的第5次Lab是一个mini projec: yesno

首先要下载并编译Kaldi，安装的过程不是我的学习重点，可以先参考[Kaldi的下载安装与编译]([https://blog.csdn.net/snowdroptulip/article/details/78896915](https://blog.csdn.net/snowdroptulip/article/details/78896915)
)，在漫长的编译过程之后假设已经安装好了Kaldi。

## 项目目录结构
yesno项目的脚本和README都在`kaldi/egs/yesno`之下。
README.txt文件中包含数据集描述：
```shell
The "yesno" corpus is a very small dataset of recordings of one individual
saying yes or no multiple times per recording, in Hebrew.  It is available from
http://www.openslr.org/1.
It is mainly included here as an easy way to test out the Kaldi scripts.

The test set is perfectly recognized at the monophone stage, so the dataset is
not exactly challenging.

The scripts are in s5/.
```
数据脚本路径： kaldi/egs/yesno/s5。在下面执行的很多操作都可以直接调用已经写好的脚本来执行，之所以深入到具体的流程中是为了加强对ASR流程的理解。

## 下载数据集
第一步是从网络上下载数据集文件`waves_yesno.tar.gz`到s5/路径下并解压。
原始的数据是60个.wav文件。文件名是八个用下划线分隔的01组合。需要将音频数据转化成kaldi能够处理的格式。


## 转换成Kaldi能处理的格式
下载完数据集后，将数据集划分为31个训练，30个测试（数量大致相当）。在s5/下创建data文件夹，把划分好的音频文件放入train_yesno和test_yesno。

Kaldi使用以下几个文件来表示数据：
1. Text
音频的文本记录。每一个音频文件一行。格式为`<utt_id> <transcript>`。<utt_id>为音频的id，一般用不带扩展名的文件名表示。utt_id在wav.scp文件中与具体的文件映射。<transcript>是音频对应的文字。
2. wav.scp
将文件映射到唯一的utt_id。
格式为`<utt_id> <path or command to get wave file>`
第二个参数既可以是对应utt_id的音频文件路径，也可以是能够获得音频文件的指令。
3. utt2spk
对于每一个音频文件，标记是哪一个人发音的。因为yesno数据集中只有一个发音者，用global来表示所有的utt_id
文件内每一行的格式为`<utt_id> <speaker_id>`

4. spk2utt
和3反过来。文件内每一行对应一个发音者，第一个是speaker的id，后面用空格分隔开60个utt_id。格式为`<speaker_id> <utt_id1> <utt_id2> ...`

本步骤可直接调用脚本：
```bash
cd kaldi/egs/yesno/s5
local/prepare_data.sh waves_yesno
```
读了一下prepare_data.sh的脚本
```bash
#!/usr/bin/env bash

mkdir -p data/local
local=`pwd`/local
scripts=`pwd`/scripts

export PATH=$PATH:`pwd`/../../../tools/irstlm/bin

echo "Preparing train and test data"

train_base_name=train_yesno
test_base_name=test_yesno
waves_dir=$1

ls -1 $waves_dir > data/local/waves_all.list

cd data/local

../../local/create_yesno_waves_test_train.pl waves_all.list waves.test waves.train

../../local/create_yesno_wav_scp.pl ${waves_dir} waves.test > ${test_base_name}_wav.scp

../../local/create_yesno_wav_scp.pl ${waves_dir} waves.train > ${train_base_name}_wav.scp

../../local/create_yesno_txt.pl waves.test > ${test_base_name}.txt

../../local/create_yesno_txt.pl waves.train > ${train_base_name}.txt

cp ../../input/task.arpabo lm_tg.arpa

cd ../..

# This stage was copied from WSJ example
for x in train_yesno test_yesno; do
  mkdir -p data/$x
  cp data/local/${x}_wav.scp data/$x/wav.scp
  cp data/local/$x.txt data/$x/text
  cat data/$x/text | awk '{printf("%s global\n", $1);}' > data/$x/utt2spk
  utils/utt2spk_to_spk2utt.pl <data/$x/utt2spk >data/$x/spk2utt
done
```
## 建立词典


对于当前项目，我们只有两个词 Ken(yes) 和 Lo（no)。但是在真实的语言中，词的数量不可能这么少，并且还有停顿和环境噪声。kaldi将这些非语言的声音称作slience（SIL）。
加上SIL一共需要三个词来表示当前这个yesno语言模型。

调用脚本：
```bash
local/prepare_dict.sh
```
将会在s5/data/local/dict中看到新生成的5个文件。
1. lexicon.txt
```
<SIL> SIL
YES Y
NO N
```
2. lexicon_words.txt
比1少第一行
3. nonsilence_phones.txt
```
Y
N
```
4. silence_phones.txt
```
SIL
```
5. optional_silence.txt
和4一样

这个脚本本身只是将input文件夹下面的lexicon_nosil.txt，lexicon.txt， phones.txt复制到dict所在目录，并且加上SIL。


## 语言模型

接下来要做语言模型。
项目提供了一个一元的语言模型。然而我们需要将这个模型转换成一个WFST（一种有穷自动机）
执行:
 ```bash
utils/prepare_lang.sh --position-dependent-phones false data/local/dict/ "<SIL>" data/local/lang/ data/lang
local/prepare_lm.sh
 ```

prepare_lang.sh的开头注释如下：
```
# This script prepares a directory such as data/lang/, in the standard format,
# given a source directory containing a dictionary lexicon.txt in a form like:
# word phone1 phone2 ... phoneN
# per line (alternate prons would be separate lines), or a dictionary with probabilities
# called lexiconp.txt in a form:
# word pron-prob phone1 phone2 ... phoneN
# (with 0.0 < pron-prob <= 1.0); note: if lexiconp.txt exists, we use it even if
# lexicon.txt exists.
# and also files silence_phones.txt, nonsilence_phones.txt, optional_silence.txt
# and extra_questions.txt
# Here, silence_phones.txt and nonsilence_phones.txt are lists of silence and
# non-silence phones respectively (where silence includes various kinds of
# noise, laugh, cough, filled pauses etc., and nonsilence phones includes the
# "real" phones.)
# In each line of those files is a list of phones, and the phones on each line
# are assumed to correspond to the same "base phone", i.e. they will be
# different stress or tone variations of the same basic phone.
# The file "optional_silence.txt" contains just a single phone (typically SIL)
# which is used for optional silence in the lexicon.
# extra_questions.txt might be empty; typically will consist of lists of phones,
# all members of each list with the same stress or tone; and also possibly a
# list for the silence phones.  This will augment the automatically generated
# questions (note: the automatically generated ones will treat all the
# stress/tone versions of a phone the same, so will not "get to ask" about
# stress or tone).
```
通过阅读脚本和脚本中的注释。可以知道`prepare_lang.sh`的用法
```
Usage: utils/prepare_lang.sh <dict-src-dir> <oov-dict-entry> <tmp-dir> <lang-dir>
e.g.: utils/prepare_lang.sh data/local/dict <SPOKEN_NOISE> data/local/lang data/lang
--position-dependent-phones (true|false)        # default: true; if true, use _B, _E, _S & _I
```
<dict-src-dir>是我们在上一部分生成的词典目录。需要包含lexico.txt，extra_questions.txt，nonsilence_phones.txt，optional_silence.txt  silence_phones.txt。
`position_dependent_phones`参数为false，导致解码后不能算出单词边界。很多的脚本，特别是评分脚本将不能正常运行。
第二个指令将语言模型转换成G.fst格式并保存在data/lang_test_tg 目录下。
这个脚本的核心内容是调用了arpa2fst和fstisstochastic，再创建了G.fst之后检查是否有空字符（<s>,</s>之类的）的循环。
```
  arpa2fst --disambig-symbol=#0 --read-symbol-table=$test/words.txt input/task.arpabo $test/G.fst

  fstisstochastic $test/G.fst
```
[arpa2fst 原理详解](https://blog.csdn.net/yutianzuijin/article/details/78756130)
> arpa文件可以很容易地表示任意n-gram语言模型，不过在实际中n通常等于3、4或者5。arpa文件的每一行表示一个文法项，它通常包含三部分内容：probability word(s) [backoff probability]。probability表示该词或词组发生的概率，word(s)表示具体的词或者词组。backoff probablitiy是可选项，表示回退概率。

在yesno这个toy project中只使用1元的语言模型。对应的arpa文件在`input/task.arpabo`
```
\data\
ngram 1=4

\1-grams:
-1	NO
-1	YES
-99 <s>
-1 </s>

\end\
```
fstisstochastic命令则如同其名字的含义一样，用来检查G.fst是否是随机的。
在`s5/data/lang`目录下会出现：
1. `phones.txt`:将phone转换成数字。其中#0，#1是空字，用来表示句子的开头和结尾。<eps> 是一个特别的含义表示这个弧上没有符号。
```
<eps> 0
SIL 1
Y 2
N 3
#0 4
#1 5
```
2. `words.txt`
```
<eps> 0
<SIL> 1
NO 2
YES 3
#0 4
<s> 5
</s> 6
```
3. `L_disambig.fst, L.fs`t: the dict can be recognized by Kaldi
4. `topo`: phone states transition(HMM)
5. `oov`: out of vocabulary. 仅包含`<SIL>`
6. ` phones`: some information about phones

## 特征提取和训练
MFCC特征提取和GMM-HMM建模

提取梅尔倒谱系数
```
steps/make_mfcc.sh --nj $num $input_dir $log_dir $output_dir
```
[语音信号处理（二）—— MFCC详解](https://zhuanlan.zhihu.com/p/60371062)
梅尔倒谱系数是一种非线性的时频表示法，其应用了人耳对低频声音的听觉敏感度更高的原理。

接着正则化倒谱特征
```
steps/compute_cmvn_stats.sh 
utils/fix_data_dir.sh $input_dir
```
[Kaldi中的特征提取(二）- 特征变换](http://placebokkk.github.io/kaldi/2019/08/05/asr-kaldi-feat2.html)
对训练集和测试集做同样的操作。
这里我采用的参数是 $num=1 $output_dir=mfcc
那么结果将会保存在mfcc文件夹下。
cmvn_test_yesno.ark  cmvn_test_yesno.scp  cmvn_train_yesno.ark  cmvn_train_yesno.scp。
ark文件包含特征向量（用cat打开是乱码），scp文件是关系文件，从发音者到ark文件的对应。
也可以直接跑脚本：
```
num=1 
output_dir=mfcc
for x in train_yesno test_yesno; do
  steps/make_mfcc.sh --nj 1 data/$x exp/make_mfcc/$x mfcc  
  steps/compute_cmvn_stats.sh data/$x exp/make_mfcc/$x mfcc        
  utils/fix_data_dir.sh data/$x 
done 
```
## 单音素模型训练
```
steps/train_mono.sh —nj $N —cmd $MAIN_CMD $DATA_DIR $LANG_DIR $OUTPUT_DIR
```
参数说明：
—nj：job的数量，来自同一个speaker的语音不能并行处理。所以在本项目中只能选择为1。
—cmd：为了使用本机的资源，调用”utils/run.pl”

运行脚本：
```
train_cmd="utils/run.pl" steps/train_mono.sh --nj 1 --cmd "$train_cmd" \   
--totgauss 400 \   
data/train_yesno data/lang exp/mono0a
```
到现在我们已经完成了模型的训练。

## 解码和测试

接下来用测试集来验证一下模型的准确与否。
第一步是创建一个全连接的FST网络。
```
utils/mkgraph.sh data/lang_test_tg exp/mono0a exp/mono0a/graph_tgpr
```
这个指令背后的脚本很长。基本上做的事情是创建以一个HCLG（HMM+Context+Lexicon+Grammer）的解码器并保存在exp/mono0a/graph_tgpr中。
还记得我们的每一条语音都是8个连续的Ken或Lo吗(Ken，Lo是希伯来语的yes no）?训练好的模型的工作就是找出这8个词的顺序。
`steps/decode.sh [options] <graph-dir> <data-dir> <decode-dir>`用来寻找每一个测试音频的最佳路径

```
decode_cmd="utils/run.pl" steps/decode.sh --nj 1 --cmd "$decode_cmd" \
exp/mono0a/graph_tgpr data/test_yesno exp/mono0a/decode_test_yesno 
```
最后是查看结果的环节。
在decode.sh内部最后会调用score.sh，这个脚本则生成预测的结果并且计算测试集Word error rate（WER）。  
调用下列命令可以看到最好的效果：
```
for x in exp/*/decode*; do [ -d $x ] && grep WER $x/wer_* | utils/best_wer.sh;
done
```
```
%WER 0.00 [ 0 / 232, 0 ins, 0 del, 0 sub ] exp/mono0a/decode_test_yesno/wer_10_0.0
```
解读一下结果，花了很长时间才弄懂这些东西是什么意思。
WER后跟着的0.00是说字的错误率为0，即准确率为100%。
测试集一共29条音频，每条音频有8个字（单音素的字）。一共232个字。
参考Stanford的cs224s-17.lec04.pdf

![WER的计算方法](https://upload-images.jianshu.io/upload_images/4328467-dc5c0fcaf24b3750.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/4328467-1c85e0672f7772cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而wer结果文件中的10和0.0分别是lmwt(Language Weight)和wip(word insertion penalty，词插入惩罚)。lmwt用来平衡LM（语言模型）在多大程度上帮助AM（语音模型）![LMWT用途 
https://skemman.is/bitstream/1946/31280/1/msc_anna_vigdis_2018.pdf](https://upload-images.jianshu.io/upload_images/4328467-3e2341231bb987fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
至于[wip](https://www.zhihu.com/question/31416764/answer/353868805)：
> word insertion penalty, 简写WIP, 是HMM识别匹配过程中用于设置句长的一个参数，可以用来调节生成句子中的单词个数，当前主流的语音识别系统主要采用的都是音素识别，即根据单词的音标而不是单词来进行匹配，这就导致了，在识别过程中，可能很难确定单词的gap，如果让系统自由识别，根据参数初始化的模型来进行匹配的话有可能会生成一些诡异的由长单词构成的句子，或者有很多短单词构成的句子，这些匹配率很低的句子对HMM参数的优化作用很小，同时也很大概率会导致学习速率奇慢或者局部最优解这样的问题，所以通过设置这个参数来得到一些更符合语义结构的生成片段。

最后，我们调用的所有脚本都在run.sh中。