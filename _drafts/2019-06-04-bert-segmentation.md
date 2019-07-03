---
layout: post
title:  "[DRAFT-URL WILL CHANGE] How morphological is BERT's segmentation?"
tags: [ "BERT", "morphology", "segmentation" ]
author: "Judit Ács"
email: "judit@sch.bme.hu"
date:   2019-06-04 11:40
---

Since its release 8 months ago, Google AI's
[BERT](https://github.com/google-research/bert) has met enormous success in a
large number of applications and garnered over 600 citations.
Two English models aside, a multilingual version was released as well, which
uses a common wordpiece vocabulary for over 100 languages.

In my previous [post](/2019/02/19/bert-tokenization-stats.html) I explored
BERT's wordpiece vocabulary and found that the distribution of the units is
skewed towards languages that use the Latin script, most notably English.
[Marianne Stecklina's Medium
article](https://medium.com/omnius/hallo-multilingual-bert-c%C3%B3mo-funcionas-2b3406cc4dc2)
provides more insight into this.  In this post I continue this exploration by
examining how well do the segmentations correspond to morphological
segmentation.


*Morphological segmentation* is the task of recovering subword units with
individual meaning. We call these units *morpemes*. An English example looks
like this [rephrase]: `tactlessness = tact + less + ness`.
It is often assumed that morphemes can be recovered by cutting the word at
character boundaries but this is not necessarily true (`flies = fly + s`). We
will stick to strict character boundaries for now, since BERT segments that way
too. Morphological segmentation is advantageous since the segments are
meaningful on their own and sharing a representation may benefit the model.
Imagine using a common representation for the plural *s* in English shared
among `cat+s` and `dog+s`. In reality, BERT handles these seemingly randomly as
illustrated in this table for *dog* and *cat* in English, French and Hungarian,
all of which are fairly regular in this case:

| Singular | Wordpieces | Plural | Wordpieces |
| ---- | :--- | :---- | :---- |
| dog | `['dog']` | dog<span style="color: red">s</span> | `['dogs']` |
| cat | `['cat']` | cat<span style="color: red">s</span> | `['cat', '##s']` |
| chien | `['chien']` | chien<span style="color: red">s</span> | `['chien', '##s']` |
| chat | `['chat']` | chat<span style="color: red">s</span> | `['chat', '##s']` |
| kutya | `['ku', '##tya']` | kuty<span style="color: green">á</span><span style="color: red">k</span> | `['ku', '##ty', '##ák']` |
| macska | `['mac', '##ska']` | macsk<span style="color: green">á<span style="color: red">k</span> | `['mac', '##ská', '##k']` |

To gain more insight [rephrase], I conducted more systematic experiments but
first let's talk about the data available for these tasks.

## Data

Gold standard morphological segmentation is only available for a few languages,
most notably the ones that were covered in
[MorphoChallenge](http://morpho.aalto.fi/events/morphochallenge/), a shared
task aiming at exactly this. They released small datasets of gold standard
segmentation for English, Finnish, German and Turkish. Furthermore, Indonesian
and Zulu datasets were released [in this
work](https://www.aclweb.org/anthology/K15-1017). I also generated Hungarian
segmentations using the [e-magyar](https://e-magyar.hu/en) toolkit and the
[Helsinki Finite-State Technology](http://hfst.github.io/).

These languages with the exception of English, are mostly agglutinative, i.e.
morphemes are mostly unchanged when they are composed into words. This is
hardly surprising considering that morphological segmentation makes little
sense if the morphemes cannot be recovered through segmentation.  English is an
exception because it features very little morphology compared to the other 6.

[TODO keep this table?]

| Language | Language family | Morphology type | Word types |
| :---- | :---- | ----: | ----: |
| English | IE, Germanic | mixed, poor | 1679 |
| Finnish | Uralic, Finnic | agglutinative | 1835 |
| German | IE, Germanic | mostly agglutinative | 1751 |
| Hungarian | Uralic, Ugric | agglutinative | 5004 |
| Indonesian | Austronesian, <br>Malayo-Sumbawan | agglutinative | 3496 |
| Turkish | Altaic, Turkic | agglutinative | 652 |
| Zulu | Niger-Congo, Bantu | agglutinative | 10040 |

<br>
All experiments from here on are performed on word *types*.

An Crubadan [TODO]

## Methods

I will compare 4 methods for segmentation: BERT's tokenizer, byte-pair encoding,
Morfessor and random. BERT's tokenizer uses a shared vocabulary for 104
languages and consecutively finds the longest wordpiece aligned to the left of
the unsegmented word. Although BERT tokenizes sentences as well, my data is
type-level, so I'll apply BERT on types as well.

Byte-pair encoding is a simple compression scheme that consecutively merges
frequent character bigrams and replaces them with a single new symbol. By doing
this iteratively, frequent character n-grams are encoded as a single symbol.
BPE is often used in neural machine translation to limit the size of the input
vocabulary (for example 30,000 BPE units). First, the units are learned on the
input data, then they're applied to segment the data into known units. BERT's
WordPiece tokenizer is a generalization of BPE. I used my [own
implementation](https://github.com/juditacs/bpe) for BPE. 

[Morfessor](http://morpho.aalto.fi/projects/morpho/) is an unsupervised
segmenter developed for concatenative morphologies. It uses the Minimum
Description Length criteria which balances the size of the morpheme lexicon and
the size of the encoding to find optimal segmentations. Morfessor and its
derivatives dominated the MorphoChallenge shared tasks. The size of the lexicon
(number of distinct segments) can be controlled via a hyperparameter.

Morfessor and BPE were trained on word frequency lists from the [An
Crubadan](http://crubadan.org/) project. First, I segmented these word lists
with BERT, then I trained the other methods to generate roughly the same amount
of distinct segments as BERT did. Finally, I segmented the smaller datasets
that had gold or silver segmentation.

My final method is a random segmenter that picks character boundaries as
segment boundaries with uniform distribution. I picked the number of output
segments from the same segment distributions that BERT generates, for example
if BERT generates 3 segments with 30% probability, the random segmenter also
generates 3 segments in 30% of the time.

## Evaluation metrics

The simplest evaluation metric is word accuracy, the proportion of words
segmented correctly. However, this metric ignores partially correct solutions,
hence boundary-level metrics such as $F_1$ TODO are usually more common. 

## Results

introduce evalutation metrics

graphs

![segmentation_accuracy](/assets/bert_segmentation/bar_word_accuracy.png)

![segmentation_f1](/assets/bert_segmentation/bar_f1.png)

![segmentation_J](/assets/bert_segmentation/bar_jaccard.png)

![segmentation_intact](/assets/bert_segmentation/bar_intact.png)

## Conclusion

BERT's segmentation is hardly morphological but this does not imply that it sucks, that should be evaluated in downstream tasks

Mention Cambridge paper
