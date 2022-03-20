+++
title = "Building a Search Engine Indexer"
date = 2022-03-19T18:26:21+01:00
draft = true
tags = ["indexer"]
+++

Many of us use search engines in our daily life, and have never stopped to wonder how they work.

I got curious and started looking for information how they function so i can build mine, here i have documented what i learnt.

A **Search Engine** is a software that helps retrieve information using keywords or phrases, and its primary functions are **Crawling**, **Indexing** & **Ranking**.

This article would be focusing on **Indexing** & **Ranking**.

## Indexing

This is the collecting, parsing, organizing and storing of content to allow fast and relevant information retrieval.

The purpose of computing an index is to optimize speed and performance in finding relevant documents for a search query. Without an index, the engine would scan every document in the corpus, which would require considerable time and computing power.

There are various structures that can be used to represent indexes:

- [Suffix Tree](https://en.wikipedia.org/wiki/Suffix_tree)
- [Citation Index](https://en.wikipedia.org/wiki/Citation_index)
- Inverted Index, etc

We would be exploring the **Inverted Index** because it easy to understand, develop and It is also the most popular data structure used in document retrieval systems.

### What is an Inverted Index?

It is a data structure that provides a mapping between terms (a word) and their location of occurrence in a text collection. Other features like ranked retrievals, spell correction, and many others can be used to extend the system. Functions similar to the glossary or word index at the back of a book.

[Image of Book Here]

There are two main variants of inverted indexes:

- **record-level inverted index** (or inverted file index or just inverted file) contains a list of references to documents for each word.
- **word-level inverted index** (or full inverted index or inverted list) additionally contains the positions of each word within a document.

The **word-level inverted index** provides more functionality (like phrase searches), but needs more processing power and space to be created.

#### Basic Structure

Let us assume our system needs to index a collection with four documents:

- **Doc1:** Take me to the heaven
- **Doc2:** Hey Jude, don't make it bad
- **Doc3:** She is buying a stairway to Heaven
- **Doc4:** Welcome to the Hotel California Such a lovely place

The structure would look like this:

```mysql
+-------------+------------+
|    Term     | Documents  |
+-------------+------------+
| buying      | Doc3       |
| carlifornia | Doc4       |
| welcome     | Doc4       |
| hotel       | Doc4       |
| heaven      | Doc1, Doc3 |
| jude        | Doc2       |
| ...         |            |
+-------------+------------+
```

Terms in this index are stored in lowercase because queries need to be case-insensitive.

The main components of an Inverted Index are **Dictionary** and **Posting Lists**. For each term in a text collection, there is a posting list that contains information about the term's occurrence in the provided collection.

#### Dictionary

This is an abstract data structure which functions essentially as a lookup table, similar to a hash map used in programming languages ex. a `dictionary` in Python or `map` in Go. When given a query we first check if it exists in our vocabulary before grabbing the corresponding postings.
We can implement the dictionary with different methods like using:

1. Sort-Based Dictionary
2. Hash-Based Dictionary

The different methods listed above do have their pros and cons but for simplicity, this article would focus on the hash-based method.

```go
dictionary := map[string]PostingList{
  "buying": [...],
  "carlifornia": [...],
  "hotel": [...],
  "heaven": [...],
  "jude": [...],
  "welcome": [...]
};
```

#### Posting Lists

This is also an abstract data structure which stores pieces of information like documents a term exists in, frequency of the term in the document, the position of the term in the document. It is usually large so mostly stored on disk instead of RAM.

_Example Structure using a [Linked List](https://en.wikipedia.org/wiki/Linked_list)_

```go
// PostingNode represents a document
type PostingNode struct {
  // document id
  docId int

  // indexes the term exists in the document
  positions []int

  // next node in the list
  next *PostingNode
}

// PostingList stores the posting list & data for a term
type PostingList struct {
  head *PostingNode
}
```

## Ranking

Like a typical search engine, its important to see relevant results and documents first because it saves us a ton of time and unnecessary scrolling looking for the relevant ones ourselves. There are tons of ranking algorithms for documents out there, but we would be implementing a simple per-term ranking algorithm called **Term Frequency, Inverse Document Frequency (tf-IDF).**

The **Term Frequency, Inverse Document Frequency (tf-IDF)** from [Wikipedia](https://en.wikipedia.org/wiki/Tf%E2%80%93idf), is a numerical statistic intended to reflect how important a word is to a document in a collection or corpus. In other words, an algorithm that helps us score words/terms in a document based on how frequently they appear in other documents.

Intuitively the algorithm works by:

- Giving a word/term a high score, If it occurs frequently in a document
- But if the word/term appears in many documents, then it is not unique. Give the word a low score.

The **tf–IDF** for a term `t` is the product of two statistics, the term's frequency and the inverse document frequency.

```txt
tfIDF = Term Frequency (tf) x Inverse Document Frequency (IDF)
```

{{<rawhtml>}}<br/>{{</rawhtml>}}

**Term Frequency (tf)** --  a measure of how many times a word/term appears in a document blob. Then normalized by dividing the count by the total number of words in a blob text.

![Term Frequency (tf)](https://latex.codecogs.com/svg.image?\LARGE&space;tf=\frac{TC}{TTC})

where,

> **TC - Term Count**, no of times the term appears in the document
>
> **TTC - Total Terms Count**, the total number of terms in the document

{{<rawhtml>}}<br/>{{</rawhtml>}}

**Inverse Document Frequency (IDF)**  --  a weight indicating how commonly a word is used. The more frequent its usage across documents, the lower its score. The lower the score, the less important the word becomes.

For any word/term `t` in the vocabulary,

![Inverse Document Frequency (IDF)](https://latex.codecogs.com/svg.image?\LARGE&space;IDF_{t}=\log_{10}\left[\frac{N}{DF_t}\right])

where,

> **N** - the total number of documents in the index
>
> **DF** - the total number of documents with term `t`

**IDF** is typically used to boost the scores of words that are unique to a document with the hope that you surface high information words that characterize your document and suppress words that don't carry much weight in a document.

For example, the word `"the"` appears in almost all English texts and would thus have a very low IDF score as it carries very little 'topic' information. In contrast, if you take the word `"coffee"`, while it is common, it's not used as widely as the word `"the"`. Thus, `"coffee"` would have a higher IDF score than `"the"`.
