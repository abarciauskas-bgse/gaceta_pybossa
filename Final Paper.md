
# Redundancy in Government Documents

## Abstract

E-government initiatives offer unprecedented transparency into government proceedings. But parsing and gleaning meaning from a large corpus is often intractable for the average citizen. Furthermore, the use of "administrative jargon" creates a barrier to understanding and parsing by the general public. Exploiting redundancy in these texts may facilitate summarization, insights and visualition of these texts.

In what follows, a simplifying assumption is made that measures of similarity can be used to signal redundancy, where redundancy is considered as a more strict definition of similarity. For example, "my dog ate my homework" and "my assignment was eaten by a canine" are both similar and redundant, whereas the former is completely similar, in one sense, to "My homework is to write about what my dog ate" but not redundant.

Given the relationship between similarity and redundancy, from the evaluation of similarity measures, a threshold for  redundancy may be gleaned from human experts. In other words, the threshold for some similarity measuring signaling redundancy is a heuristic to be determined.

## Data

Two sources of data were used for this project:

1. Minutes of the Federal Open Market Commission (FOMC)
2. Barcelona's Municipal Gazette

For both data sources, the available data was subset by a time window to make computation tractable (years in the case of FOMC minutes and months for Gaceta data). Corpii were composed of all documents available in the time window.

#### FOMC data

[Source](http://stanford.edu/~rezab/useful/fomc_minutes.html)

Minutes of FOMC meetings are available in plain text from 1967 to 2008. Each year there are about 8 reports released.

The period composing a corpus for FOMC data for this project was a year, so a corpus comprised of 8 documents.

#### Barcelona's Municipal Gazette

[Source](https://w33.bcn.cat/GasetaMunicipal/Inici?lang=EN)

PDFs of the Municipal Gazette are available from the years from 2000-2016. The number of publications per year varies, but 560 total are available. These 560 pdf documents were parsed into 13,116 sub-documents.

Sub-documents are extracted from PDF documents in 2 phases: First, raw text is extracted programmatically. Second, a program defined by a native Catalan speaker using heuristics parses out sub-documents composing the main bodies of text from the original.

The PDF documents included long tables of contents and data tables. In the second phase, these non-textual objects are omitted and resulting sub-documents are comprised of main sectiosn from the original document body, such as "Acords" (i.e. "agreements") and various commissions' reports.

The period composing a corpus for this project was a month-year, selected such that again the size of the corpus was computationally manageable.

## Question

### Is redundancy a tool for summarization?

E-government initiatives offer unprecedented transparency into government proceedings. But often their size and use of "administrative jargon" creates a barrier to understanding and parsing by the general public. Exploiting redundancy in these texts may facilitate the initiative to provide insight via summarization and visualization of these texts.

Citizens should have transparency in government, but transparency is meaningless when its manifestation is only in undigestable amounts of text. How is relevant information stored in these corpii?

This project explores the hypothesis repetition conveys information about how a body that produced a corpus operates (e.g. named entity relationships) and or conveys summary information.

The field agrees that redundancy and similarity are theoretically valuable sources of information about a corpus:

This is the motivation for evaluating redundancy in the current effort. Additional motivations are included as inspirational:

> Sentence similarity is considered the basis of many natural language tasks such as information retrieval, question answering and text summarization. ([A Comprehensive Comparative Study of Word and Sentence Similarity Measures](https://www.researchgate.net/publication/294873785_A_Comprehensive_Comparative_Study_of_Word_and_Sentence_Similarity_Measures))

_________

> ..redundancy can be exploited to identify important and accurate information for applications such as summarization and question answering ([Sentence Fusion for Multidocument News Summarization](http://www.mitpressjournals.org/doi/pdf/10.1162/089120105774321091)).

_________

> ...most applications based on Twitter share the goal of providing tweets that are both informative and diverse... to keep a high level of diversity, redundant tweets should be removed from the set of tweets displayed to the user ([Linguistic Redundancy in Twitter](http://www.aclweb.org/anthology/D11-1061.pdf)).

_________

> ...from a computational linguistic point of view, the high redundancy in micro-blogs gives the unprecedented opportunity to study classical tasks ... on very large corpora characterized by an original and emerging linguistic style, pervaded with ungrammatical and colloquial expressions, abbreviations, and new linguistic forms  ([Linguistic Redundancy in Twitter](http://www.aclweb.org/anthology/D11-1061.pdf)).

_________

> O'Shea et al. applied text similarity in Conversational Agents, which are computer programs that interact with humans through natural language dialogue ([Text Similarity using Google Tri-grams](https://web.cs.dal.ca/~eem/cvWeb/pubs/2012-Aminul-CAI.pdf)).

## Extracting Content

Pre-processing of the data was done in Java using the [Freeling NLP](https://github.com/TALP-UPC/FreeLing) library and Porter Stemmer algorithm. Data were stored in PostgreSQL databases `fomc` and `gaceta` each having tables `corpii`, `processed_documents` and `alignments`. See [`schema.sql`](https://github.com/abarciauskas-bgse/gaceta/blob/master/schema.sql) for detail on these database schema.

To populate the `processed_documents` table, the following steps were taken:

1. The codebase is updated to reflect the language and time window\*.
2. Each document in the current "corpus" (that is, each document corresponding to the time window of interest) was read from local storage
3. The entire body of text is parsed into sentences, each sentence is a document in the corpus. Sentences are required to be 4 terms long.
4. Each term in a document is:
   1. removed if punctuation or a stopword ([stopwords provided by upc.edu](http://www.cs.upc.edu/~padro/index.php?page=nlp))
   2. replaced with a tag if it is identified by the morphological analyzer as a quantity (replaced with 'QUANT'), date ('DATE'), miscellaneous ('MISC'), person ('PER'), location ('LOC'), organization ('ORG') or other proper noun ('NP')
   3. Stemmed using the PorterStemmer algorithm\*\*.
5. Terms from all documents (e.g. sentences) are used to compoce a `TermVector` which is stored in the `corpii` table. Document term and tf-idf matrices are computed for the current corpus, and each document's respective vector in the tf-idf matrix is stored as it's `TfIdfVector` in the `processed_documents` table.

Once all documents have been stored, similarity measures are calculated for pairs of documents. But more on this below, following a **Review of Current Methods**.

\* *see [the gaceta codebase README](https://github.com/abarciauskas-bgse/gaceta/blob/master/README.md) for more details*

\*\* *In the case of Catalan, this algorithm should not be used and Freeling's getLemma should do the trick, but it is hard to tell if that is working for an english speaker. Changing the locale for english exposed some error in the freeling installation, so the PorterStemmer was used as a workaround for not having a working version of Freeling for english documents. This doesn't seem to be a problem in identifying entities as described in step 4.2. 

## A Review of Current Methods

To start the task of analyzing redundancy, a review of existing methods informed the path forward.

### Methods to Measure Similarity and Redundancy

In the existing literature, measurements of similarity have been separated into **corpus-**, **knowledge-**, and **hybrid-based** methods. Hybrid methods are excluded from the current review.

The practical difference between corpus and knowledge-based methods is the corpus based depends on word frequences from a specific corpus. A restriction on corpus-based methods is they are quite domain-dependent and often do not generalize outside of a given corpus. This could pose a potential problem for the current efforts if required to measure redundancy across e-government initiatives, but this is not a current requirement, so this limitation is acceptable.

All methods listed below are included given their pertinence to the current problem, with the exception of methods listed in **Of Interest**.

#### Corpus-Based Word Similarity

The bag-of-words (BOW) method is often used as a baseline measurement of similarity between documents. Given a document-term matrix, taking the dot product or cosine of the dot-product between two columns (e.g. documents) gives a BOW-based similarity score of those two documents. The same method can be followed for the tf-idf version of this matrix.

[Latent Semantic Analysis][Latent Semantic Analysis] (LSA) measures the similarity between words using a word-count per document (e.g. words x documents, or transpose of the document-term matrix) matrix and computing the cosine of the dot product between 2 rows. This within-corpus word similarity measure will enrich measurements of similarity when comparing documents in methods for computing sentence-based similarities in what follows.

#### Knowlege-Based Word Similarity

WordNet bag-of-words (WBOW) is a "knowledge-based" version of Latent Semantic Analysis and is frequently used to enrich measurements of similarity in texts. WordNets are human-generated lexicons and thus do not require the pre-computation or corpus-dependency of LSA. WordNets are popular but may be limited in depth. It will be interesting to see what is available for Catalan.

#### Knowledge-Based Document Similarity

Knowledge-based document similarity measures listed in [Atoum](https://www.researchgate.net/publication/294873785_A_Comprehensive_Comparative_Study_of_Word_and_Sentence_Similarity_Measures) use a knowledge-based measurement of word similarity within a document and some quantification for document structure. For example, measurments composed of [WBOW plus part-of-speech (POS) tree kernels](http://ieeexplore.ieee.org/xpl/) or [POS tags](http://www.sciencedirect.com/science/article/pii/S0957417410011875?np=y). Others are listed in [Atoum Section 2.2.2](https://www.researchgate.net/publication/294873785_A_Comprehensive_Comparative_Study_of_Word_and_Sentence_Similarity_Measures). Theses methods demonstrated poor results or were not evaluated in [Atoum](https://www.researchgate.net/publication/294873785_A_Comprehensive_Comparative_Study_of_Word_and_Sentence_Similarity_Measures), and some of the more attractive versions are not available for review. For these reasons, focus will be on corpus-based methods (and possibly hybrid-based methods later on).

#### Corpus-Based Document Similarity

Corpus-based measures of document similarity rely on string similarity, string edit distance and word orders. Other common methods are the [edit-distance](https://en.wikipedia.org/wiki/Edit_distance) and [Smith-Waterman Alignment](https://en.wikipedia.org/wiki/Smith%E2%80%93Waterman_algorithm). These methods will be used as baselines for more advanced methods, but also may provide valuable insights.

In [Atoum](https://www.researchgate.net/publication/294873785_A_Comprehensive_Comparative_Study_of_Word_and_Sentence_Similarity_Measures), the highest-performing method was the [Google Tri-Gram](https://web.cs.dal.ca/~eem/cvWeb/pubs/2012-Aminul-CAI.pdf) approach. This approach calculates a word-similarity metric using trigrams and then uses it in a subsequent text similarity metric. In essence, this metric evaluates the similarity of word `w_a` and `w_b` by measuring the frequency of tri-gram instances containing `w_a` and `w_b` in positions 1 and 3 of the trigram.

Tree-based measurements leverage a tree data-structure representation of a document (i.e. sentence). In [Linguistic Redundancy in Twitter](http://www.aclweb.org/anthology/D11-1061.pdf) the most successful formulation was a combination metric using WBOW and the Syntatic First-Order Rule Content Model (FOR). The FOR feature space introduced by [Zanzotto and Moschitti](http://www.aclweb.org/anthology/P06-1051) constructs features as a pair of syntatic tree fragments augmented with variables which are evaluated for similarity.

[Simfinder](http://www.mitpressjournals.org/doi/pdf/10.1162/089120105774321091) also uses sentence syntax trees to compte sentence similarity, without expectation on their complate alignment.

## Methodologies (continuation of Extracting Content)

Two methodologies were developed for this problem, building off the review above:

1. The Needleman-Wunsch global sequence alignment algorithm
2. Cosine similarity and clustering based on the tf-idf matrix of a document

### 1. The Needleman-Wunsch global sequence alignment algorithm

The Needleman-Wunsch (NW) Algorithm surfaces the best global alignment between 2 sequences of text. The Smith-Waterman (SW) alignment is a variation on Needleman-Wunsch which finds the best local alignment between 2 sequences of text. Both compute a scoring matrix in the same forward pass of the algorithm, with NW using negative values to indicate a gap penalty. SW does not include negative values, instead inserting 0's for gaps. In the backward pass to find the best global or local alignment, NW starts from the terminal cell and finds the highest scoring alignment for the whole strings, whereas SW starts from the cell with the highest score and returns the equivalent of the longest common subsequences between the 2 strings. 

NW has been implemented along with a `printAligment` function which returns an alignment pattern:

```
[|, de, |, |, |, |, |, 2000, |, per, |, |, |, |, |, |, |, |, |, |, |, |, |, |, |, |, --, --, |, |, |, |, per, ...]
[|, de, |, |, |, |, |, 2000, |, per, |, |, |, |, |, |, |, |, |, |, |, |, |, |, |, |, handbol, per, ...]
```

##### Implementation

The NW algorithm is implemented in the codebase [here](https://github.com/abarciauskas-bgse/gaceta/blob/master/src/com/gaceta/Alignment.java#L19-L58). Each score is stored in the `alignments` table along with the corresponding document ideas.

Below is the distribution of these scores for an FOMC corpus.

![hist](/Users/aimeebarciauskas/IdeaProjects/gaceta/python_scripts/nw_distribution_fomc.png)

##### Evaluation

The human intelligence platform [pybossa](http://pybossa.com/) was used to find a threshold qualifying 2 pairs of statements as redundant. Pybossa is a platform for crowdsourcing classification tasks which require human intelligence.

The projects may be found:

* [crowdcrafting.org/project/fomc/](crowdcrafting.org/project/fomc/)
* [crowdcrafting.org/project/gaceta/](crowdcrafting.org/project/gaceta/)

And corresponding related codebases:

* [ADDME]
* [ADDME]

Using human decisions on redundancy of pairs of sentences, a match is made with the similarity score determined by an algorithm. This will determine a threshold. Pairs of sentences with a score above this threshold will be classified as redundant.

For the FOMC data, 6 persons were polled on 40 NW alignment scores selected from 0 to 4 deviations from the mean NW score. The following plots show the distributions of scores for those pairs of sentences marked redundant vs not redundant

![hist](/Users/aimeebarciauskas/IdeaProjects/gaceta/python_scripts/marked_redundant_hist.png)
![hist](/Users/aimeebarciauskas/IdeaProjects/gaceta/python_scripts/marked_not_redundant_hist.png)

It is fairly clear that redundant documents require an alignment score of at least 0.

##### References

* [Needleman-Wunsch Algorithm (SlideShare)](http://www.slideshare.net/avrilcoghlan/the-needleman-wunsch-algorithm)
* [Needleman-Wunsch Algorithm (Wikipedia)](https://en.wikipedia.org/wiki/Needleman%E2%80%93Wunsch_algorithm)
* [Smith-Waterman Algorithm (SlideShare)](http://www.slideshare.net/avrilcoghlan/the-smith-waterman-algorithm)
* [Smith-Waterman Algorithm (Wikipedia)](https://en.wikipedia.org/wiki/Smith–Waterman_algorithm)
* [What is the difference between local and global sequence alignments?](http://biology.stackexchange.com/questions/11263/what-is-the-difference-between-local-and-global-sequence-alignments)

### Cosine similarity and clustering based on the tf-idf matrix of a document

Cosine similarity captures the commonality in words regardless of sequence in a document, which may be valuable in summarization. Cosine similarity is the measure of the angle between two vectors and was measured between each pair of tf-idf vectors within a corpus and stored in the `alignments` table.

##### Implementation

The implementation can be found in the codebase [here](https://github.com/abarciauskas-bgse/gaceta/blob/master/src/com/gaceta/Utils.java#L43-L53).

##### Evaluation

Are there pairs with low NW scores but high cosine similarity? If so, it suggests there may be redundant concepts which are not captured in alignment scores.

![hist](/Users/aimeebarciauskas/IdeaProjects/gaceta/python_scripts/nw_scores_v_cosine_sims_fomc.png)

In the scatterplot above, there are many cosine similarities which correspond to extremely low NW scores. This suggests there may be some value to looking documents with similar words and not only similar sequences of words.

## Addressing Your Question

Can redundancies offer an opportunity for summarization? To address the question, each similarity measure was clustered and clusters analyzed.

### Clustering based on NW Score

A NW score of 0.0 was found to be a reasonable threshold for pairs of redundant statements, based on human intelligence tasks as described in **Methodologies**. Restricting analysis only to sentence pairs above this threshold greatly reduces the size of alignments to analyze.

Reducing the problem with this redundancy threshold is good because it creates small and self-contained clusters. However it most certainly omits a lot of information. For 2005 FOMC minutes data, there exists 523,776 pairs of sentences, however restricting the analysis to those sentences having a NW score greater than 0.0 leaves just 274 pairs.

To construct clusters of sentences, fully connected networks of sentences are constructed. A whole graph composed of all documents and alignments with NW scores greater than 0.0 is fed to an algorithm to find fully connected sub graphs within it.

For the FOMC 2005 data, the distribution of the size of each of these clusters is found in the histogram below.

![hist](/Users/aimeebarciauskas/IdeaProjects/gaceta/python_scripts/redundant_cluster_sizes_fomc.png)

Most clusters contain just a pair of sentences, but more interestingly is there are a small number of large graphs.

The content of these larger clusters is given by determining the centroids and relative term frequencies. Centroids are defined as the vertex with the greatest number of edges to other vertices in the sub-graph).

For 2005 FOMC data, the content of the centroids for the top 5 clusters is:

> The Manager also reported on developments in domestic financial markets and on System open market transactions in government securities and federal agency obligations during the period December 14, 2004 to February 1, 2005.

_________


> At the conclusion of the discussion, the Committee voted to authorize and direct the Federal Reserve Bank of New York, until it was instructed otherwise, to execute transactions in the System Account in accordance with the following domestic policy directive:
"The Federal Open Market Committee seeks monetary and financial conditions that will foster price stability and promote sustainable growth in output. To further its long-run objectives, the Committee in the immediate future seeks conditions in reserve markets consistent with increasing the federal funds rate to an average of around 3 percent." The vote encompassed approval of the paragraph below for inclusion in the statement to be released shortly after the meeting:" The Committee perceives that, with appropriate monetary policy action, the upside and downside risks to the attainment of both sustainable growth and price stability should be kept roughly equal. With underlying inflation expected to be contained, the Committee believes that policy accommodation can be removed at a pace that is likely to be measured. Nonetheless, the Committee will respond to changes in economic prospects as needed to fulfill its obligation to maintain price stability."
Votes for this action: Messrs.

_________


> There were no open market operations in foreign currencies for the Systems account in the period since the previous meeting.

_________


> At its June meeting, the Federal Open Market Committee decided to increase the target level of the federal funds rate 25 basis points, to 3 percent.

_________


> In its accompanying statement, the Committee indicated that, with appropriate monetary policy action, the upside and downside risks to the attainment of both sustainable growth and price stability should be kept roughly equal.



Not being an expert in FOMC proceedings, these look a good summary of  meetings: asessment of market conditions and implementation of monetary policy to maintain price stability through adjusments to the federal funds rate.

### Clustering based on Cosine Similarity

The cosine similarity offers a categorical view of topics covered. Instead of threshold holding redundancy at some value, a more nuanced view of the corpus can be gleaned from cluster a larger fully connected graph of sentences.

To do this clustering, the minimum k-cut karger approximation algorithm is used. This algorithm uses asymptotic theory to find the minimum cut in a graph with high probability.

A cut in a graph is a partition of vertices into 2 or more sets. A minimum cut is a cut of the graph where the number of edges across partitions is minimized. Given a fully connected graph, finding the minimum cut is a clustering algorithm which finds the [highly connected subgraphs](https://en.wikipedia.org/wiki/HCS_clustering_algorithm). Highly connected subgraphs are those where nodes in the graph are more similar than nodes.

The Karger algorithm randomly contracts edges creating supernodes until k supernodes are left. The probability of finding the minimum cut on any one iteration is very low, but the probability given many iterations is very high. 

Given `N` is the number of iterations the algorithm is run and `n` is the number of vertices in the graph, then the [probability N trials fail](https://d396qusza40orc.cloudfront.net/algo1/slides/algo-karger-analysis_typed.pdf) to find the minimum cut is:

```
(1 - 1/n**2)**N
```

##### The experiment

Using 2005 FOMC data, alignments were thresholded to 0.25 cosine similarity. This resulted in 2577 alignments. The complete graph of alignments containted 877 vertices and the largest fully connected graph from those alignments contained 835 vertices.

Unfortunately, even for this smallish graph, to run the Karger approximation to 99.9% probability of success of finding the min cut will take nearly 95 days on my poor machine. I hope to have started a new job by then.
