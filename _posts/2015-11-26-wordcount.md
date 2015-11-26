---
layout: post
title:  "Counting words in different programming languages"
tags: [ "programming", "python", "c++" ]
author: "Judit Ács"
email: "judit@sch.bme.hu"
date:   2015-11-26 19:00
---

Creating word frequency lists is an easy task in most programming languages but how easy it is exactly?
And what are the performance trade-offs?
We played around with our favorite programming languages and got suprising results.
The experiment is still going, you can participate too. And please do.

A couple weeks ago I had a homework where I had to create a word frequency list from text read from the standard input, and output the words and frequencies in decreasing order.
Ties had to be printed in alphabetical order.
This seems like a fairly easy task, I do it all the time in Python, so I was curious how easy it would be in C++.

My first version looked like this [[wordcount_map.cpp]](https://github.com/juditacs/wordcount/blob/master/cpp/wordcount_map.cpp):

{% highlight c++ %}
#include <iostream>
#include <map>
#include <set>


class F {
    public:
        bool operator()(std::pair<int, std::string> lhs, std::pair<int, std::string> rhs) {
            if (lhs.first > rhs.first) return true;
            if (lhs.first == rhs.first && lhs.second < rhs.second) return true;
            return false;
        }
};

int main() {
    std::map<std::string, int> m;
    std::string s;
    while (std::cin >> s) {
        ++m[s];
    }
    std::multiset<std::pair<int, std::string>, F> mm;
    for (auto p: m) mm.insert(std::pair<int, std::string>{p.second, p.first});
    for (auto p: mm) {
        std::cout << p.second << "\t" << p.first << std::endl;
    }
}
{% endhighlight %}

The algorithm is rather straightforward:

1. Define a counter object, it has to be addressable with arbitrary strings.
2. Count words.
3. Sort the counter by frequency.
4. Print word-frequency pairs.

Now let's see how it works.

~~~~
$ echo "alma körte szilva alma körte" | ./cpp/wc_baseline
alma    2
körte   2
szilva  1
~~~~

It does a good job counting the occurrences on 5 words (Hungarian fruit names).

How does it work on a bigger dataset?
We will get back to it later.


Since I do similar tasks in Python almost every day, I did the shortest Python implementation I could come up with.
A slightly neater version is available here: [wordcount.py](https://github.com/juditacs/wordcount/blob/master/python/wordcount_py2.py)

{% highlight python %}
from collections import defaultdict
from sys import stdin

counter = defaultdict(int)
for l in stdin:
    for word in l.split():
        counter[word] += 1
for word, cnt in sorted(counter.iteritems(), key=lambda x: (-x[1], x[0])):
    print('{0}\t{1}'.format(word, cnt))
{% endhighlight %}

Step 3 and 4 can be grouped together into a sorted iteration over the dictionary elements.

## Test data

Let's test both scripts on some real data.
I downloaded the Hungarian Wikisource XML database dump for testing and did no further preprocessing.
Although the whole file is 2,000,000 lines, I used the first 500,000 for testing.

    $ g++ cpp/wordcount_map.cpp -std=c++11 -o cpp/wc_baseline -O3
    $ time head -500000 data/huwikisource-latest-pages-meta-current.xml | ./cpp/wc_baseline > /tmp/cpp_out
    real    0m6.308s
    user    0m5.920s
    sys     0m0.480s

Now let's compare it with Python.

    $ time head -500000 data/huwikisource-latest-pages-meta-current.xml | ./python/wordcount_py2.py > /tmp/python2_out
    real    0m5.647s
    user    0m5.477s
    sys     0m0.220s

Ok, that was unexpected. Python's supposed to much slower than C++, right?
Before we dwell into the details, let's see the output.

## The output

The output is nothing surprising. The most frequent words are Hungarian function words, articles (*a, az, és*) on the top.
We also get some MediaWiki markup mixed in such as *=*.

~~~~
$ head /tmp/cpp_out 
a       126995
|       45751
az      42187
és      37088
=       22811
nem     22529
hogy    20949
A       19994
s       19042
-       17857

~~~~

The bottom of the list looks like this:

~~~~
$ tail /tmp/cpp_out 
와일드]]        1
작품]]</comment>        1
저자]]</comment>        1
태도]]  1
프롤레타리아]]  1
�</text>        1
�Bon    1
�Good   1
�Guten  1
�békés  1
~~~~

(It's clear from these few lines that the *word tokenization* is far from trivial and for real life scenarios we need more sophisticated solutions than this.)

There are actually 421,738 hapax (words that only appear once) out of 2,758,420 total words (573,090 types - i.e. unique words).
Since we require a secondary sort by words in the case of frequency ties, non-Hungarian words and invalid characters are sorted last place.

## Why is the C++ implementation slower?

Examining the data structures tells us that `std::map` is actually a red-black tree in contrast to Python's `dict` which is a hash table.
Both have their advantages but for this case insertion is particularly interesting, since we insert almost 600 thousand words.
The insertion in a red-black tree is *O(log n)*, while it is *O(1)* on average in the case of hash table.
There are just too many word types.

Fortunately the STL has several hash table-based containers such as the `std::unordered_map`, we just have to replace `std::map` [wordcount_hashtable.cpp](https://github.com/juditacs/wordcount/blob/master/cpp/wordcount_hashtable.cpp):

{% highlight c++ %}

#include <unordered_map>

// ...

int main() {
    std::unordered_map<std::string, int> m;

// ...
{% endhighlight %}


~~~~
$ g++ cpp/wordcount_hashtable.cpp -std=c++11 -o cpp/wc_hash -O3
$ time head -500000 data/huwikisource-latest-pages-meta-current.xml | ./cpp/wc_hash > /tmp/cpp2_out

real    0m4.970s
user    0m4.613s
sys     0m0.520s

$ diff /tmp/cpp_out /tmp/cpp2_out | wc -l # making sure the outputs are the same
0
~~~~

Python is finally beaten but not very much.
And it turns out that with an older g++ (mine is 5.2 on Manjaro Linux) this is not even true, but let's get back to that later.

Can we speed up C++ a bit more?

Standard C++ streams are synchronized to the standard C streams after each input/output operation.
This behavior can be disabled by setting [sync_with_stdio](http://en.cppreference.com/w/cpp/io/ios_base/sync_with_stdio) to false.

{% highlight c++ %}
std::ios_base::sync_with_stdio(false);  // call this at the beginning of main
{% endhighlight %}

Now let's try this version (see the full code [here](https://github.com/juditacs/wordcount/blob/master/cpp/wordcount_hashtable_nosync_stdio.cpp) ).

~~~~
$ g++ cpp/wordcount_hashtable_nosync_stdio.cpp -o ./cpp/wc_hash_nosync -std=c++11 -O3
$ time head -500000 data/huwikisource-latest-pages-meta-current.xml | ./cpp/wc_hash_nosync >/tmp/cpp_out

real    0m3.647s
user    0m3.273s
sys     0m0.480s

~~~~

Getting better.

Still, a few things can be improved, like using const references instead of copying or inserting the (word, frequency) pairs into a vector, then sorting the vector.
The current best version is [here](https://github.com/juditacs/wordcount/blob/master/cpp/wordcount_vector.cpp).

~~~~
$ g++ cpp/wordcount_vector.cpp -o ./cpp/wc_vector -std=c++11 -O3
$ time head -500000 data/huwikisource-latest-pages-meta-current.xml | ./cpp/wc_vector > /tmp/cpp3_out

real    0m2.644s
user    0m2.340s
sys     0m0.403s
~~~~

# Other languages

When I told my friends and collegues about this experiment, they were really enthusiastic about trying other languages and we wound up with a bunch of languages and wordcounters just waiting to be tested against each other.
In addition to Python and C++, currently we have word counters in Java, shell script, Javascript, Julia, Perl, Python3 and PHP.
I created a [GitHub repository](https://github/com/juditacs/wordcount) for this project and it was soon populated.

The current leaderboard is [here](https://github.com/juditacs/wordcount#leaderboard).

# Full results

The full test was run in a Docker container on Ubuntu with GCC 4.8, so - to my surprise - the running times were quite different too.

Full results for 500,000 lines are summarized in this table:

| Rank | Experiment | CPU seconds | User time | Memory |
| :---: | :---: | :---:|:---: | :---: |
| 1 | C++ best | 3.87 | 2.79 | 90548 |
| 2 | C++ nosync | 4.44 | 3.27 | 104268 |
| 3 | Python2 | 4.73 | 4.62 | 169524 |
| 4 | C++ hash | 5.66 | 4.47 | 109292 |
| 5 | C++ map | 6.84 | 5.65 | 110080 |
| 6 | Java | 7.18 | 7.82 | 258476 |
| 7 | Python3 | 7.41 | 7.22 | 147764 |
| 8 | PHP | 7.96 | 6.72 | 251604 |
| 9 | Shell script | 18.91 | 21.98 | 12860 |

And for the full huwikisource dump:

| Rank | Experiment | CPU seconds | User time | Memory |
| :---: | :---: | :---:|:---: | :---: |
| 1 | C++ best | 15.96 | 12.24 | 244404 |
| 2 | C++ nosync | 18.81 | 15.02 | 327184 |
| 3 | Python2 | 20.02 | 19.58 | 554352 |
| 4 | C++ hash | 24.23 | 20.45 | 343276 |
| 5 | Java | 27.37 | 31.39 | 545940 |
| 6 | Python3 | 33.06 | 32.59 | 482988 |
| 7 | C++ map | 35.1 | 31.25 | 355692 |
| 8 | PHP | 39.82 | 35.58 | 781668 |
| 9 | Shell script | 82.36 | 93.9 | 12924 |

# Experimental setup

I've never used Docker before and I decided to give it a try for this experiment.
Docker is similar to a virtual machine but much more lightweight.
You can download images or build your own and load them into Docker containers which are mini operating systems with root access for you.
Images are built by running commands read from a Dockerfile.
I'm not going to bore you with all the technical details, you can find an explanation in this [README](https://github/com/juditacs/wordcount).

I created a Dockerfile which installs the dependencies needed by this experiment.
My friends added other languages and their dependencies too, so once you build the image and load it into a container, the experiment should work without installing additional packages.

You can reproduce the whole experiment with running a few commands inside your container.

# Can you do better?

Your favorite language is missing?
Or you can do better in language X? Please do.

You're welcome to improve any of the existing scripts or provide a new one.
Please read the [wordcount](https://github.com/juditacs/wordcount) repository's README for instructions on how to install and add a new script.
If your code successfully installs and compiles with the provided Docker image and it passes the tests, run a few experiments.
If you have reasonably good results - not necessarily a new winner - please create a pull request and I'll add it to the main repository.

# Disclaimer

The point of this experiment was to compare *simple* solutions in different programming languages to a *simple* problem.
It turns out the results are not what I expected, but by no means am I trying to tell which language is better than the other.
I also do not claim that these are the best possible solutions.

# Acknowledgements

I thank the authors of all the other scripts, in alphabetical order: Patrik Braun (PHP), László Kundra (Javascript) and Dávid Nemeskey (Java and suggestions for C++).
I also thank Zoltán Czirkos, András Kornai and Gábor Borbély for useful suggestions.
