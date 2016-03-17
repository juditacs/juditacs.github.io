---
layout: post
title:  "Counting words in different programming languages II."
tags: [ "programming", "python", "c++" ]
author: "Judit Ács"
email: "judit@sch.bme.hu"
date:   2016-02-26 19:00
---


I started a word counting challenge a few months ago and it received a lot more
interest than I expected. Yes, I admit that I deliberately made the challenge
sound very easy thus more people would participate.

Currently there are 18 unique solution in 12 languages if you count Python2 and
3 separate languages. Over a dozen people contributed, I thank you all.

This has been an amazing experience for me, and I learnt a few valuable lessons
during this competition:

* I should have made the rules much clearer and much sooner.
* I should have decided on the test data sooner to avoid having 3-4 leaderboard
tables.
* I ended up adding half of the test cases later than announcing the
competition which was necessary but unfair.


# The final dataset

I used the Hungarian Wikipedia for the final experiments. Each solution was
tested on the first 5 million lines and later on the full Hungarian Wikipedia
TODO meret, sorok szama

Most of them performed well on the full dataset except:

* all Java versions seem to hang for hours and were killed (by me) after a day,
* the Javascript version runs out of memory (16GB).


# Can I still contribute?

Yes, you are welcome to contribute. However, I won't update this article with
your results. If a second wave of great solutions come, I might write a third
article.


| Rank | Experiment | CPU seconds | User time | Maximum memory |
| :---: | :---: | :---: | :---: | :---: |
| 1 | cpp/wc_v6 | 260.87 | 250.97 | 3904412 |
| 2 | python/wordcount_py2gabor.py | 333.34 | 321.56 | 3844908 |
| 3 | go/bin/wordcount | 349.26 | 332.43 | 6066928 |
| 4 | python/wordcount_py2.py | 545.38 | 529.71 | 8670208 |
| 5 | mono csharp/WordCountList.exe | 796.2 | 637.8 | 4780360 |
| 6 | perl/wordcount.pl | 881.23 | 861.38 | 6979772 |
| 7 | python/wordcount_py3.py | 909.86 | 888.65 | 7561112 |
| 8 | php php/wordcount.php | 1107.38 | 998.33 | 12467064 |
| 9 | julia julia/wordcount.jl | 1798.55 | 1763.0 | 7284708 |
| 10 | bash/wordcount.sh | 2100.96 | 2128.94 | 13768 |


