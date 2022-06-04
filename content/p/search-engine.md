+++
title = "Building a Search Engine Indexer"
date = 2022-03-19T18:26:21+01:00
draft = true
tags = ["indexer"]
+++

Everyone uses search engines frequently for many applications, whether to check live scores of an NBA game or confirm a favorite celebrity's age. In addition, there are several billion bytes of information present on the internet. So how does a search engine know which is the most suitable for your request?

So, I got curious to find out how these 'engines' provide the most relevant information to every one of our requests in fractions of a second. Here, I have documented what I learned.

# Indexing

This is the process of collecting, parsing, organizing, and storing documents to allow fast and relevant information retrieval.

Pre-computing an index aims to optimize speed and performance in finding relevant documents for a search query. Without an index, the engine would need to scan every record in the collection, which would require considerable time and computing power.

We can represent indexes with different structures; here are a few:

- [Suffix Tree](https://en.wikipedia.org/wiki/Suffix_tree)
- [Citation Index](https://en.wikipedia.org/wiki/Citation_index)
- Inverted Index, etc

We would be exploring the Inverted Index because it is easy to understand and develop. It is also the most popular data structure used in document retrieval systems.

## Inverted Index?

It is a data structure that provides a mapping between terms (a word) and their location of occurrence in a text collection. Think of it as the word index at the back of a textbook where a term references an associated page number.

{{< image
  caption="We can see words mapped to pages"
  src="https://res.cloudinary.com/dj3nunory/image/upload/bo_1px_solid_rgb:000000,r_0/v1649197385/blessing.pario.la/book-index.jpg" >}}

### Basic Structure

Let us assume our system needs to index a collection with four documents:

- **Doc1:** Take me to the heaven
- **Doc2:** Hey Jude, don't make it bad
- **Doc3:** She is buying a stairway to Heaven
- **Doc4:** Welcome to the Hotel California Such a lovely place

After breaking our document's contents into tokens, the structure would look like this:

{{< image
  src="https://res.cloudinary.com/dj3nunory/image/upload/v1654120121/blessing.pario.la/inverted-index-structure.png" >}}

There are two main variants of inverted indexes:

- **Record-Level inverted index** (or inverted file index or just inverted file) contains a list of references to documents for each word.
- **Word-Level inverted index** (or full inverted index or inverted list) additionally contains the positions of each word within a document.

The difference is that the Word-level inverted index provides more functionality (like phrase searches). Hence, it needs more processing power and space.

Let's explore the main components of an Inverted Index - **Dictionary** and **Posting Lists**.

### Dictionary

The dictionary functions as a lookup table, similar to a hash map used in programming languages. When given a search query, we first check if it exists in our vocabulary before grabbing the corresponding postings.

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

### Posting Lists

The posting list contains bits of information for the related term, everything including:

- Documents the word exists in
- Position of the word in the document
- Frequency of the word in the document, etc.

It is usually large, so we primarily store it on disk instead of RAM.

_Using a [Linked List](https://en.wikipedia.org/wiki/Linked_list):_

```go
// PostingNode represents a document
type PostingNode struct {
  // document id
  docId int

  // next node in the list
  next *PostingNode
}

// PostingList stores the posting list & data for a term
type PostingList struct {
  head *PostingNode
}
```

During pre-processing, performing transformations like [lemmatization](https://en.wikipedia.org/wiki/Lemmatisation), removing common [stop words](https://en.wikipedia.org/wiki/Stop_word), [stemming](https://en.wikipedia.org/wiki/Stemming), and many others help improve the index's quality.

```go
// common stop words
var stopWords = map[string]struct{}{
  "a": {},
  "the": {},
}

func lex(text string) []string {
  var (
		term   strings.Builder
		tokens []string
	)

	for i := 0; i < len(text); i++ {
		r := rune(text[i])

		if !unicode.IsLetter(r) && !unicode.IsNumber(r) {
			if term.Len() > 0 {
				word := term.String()

				// remove stop words
				if _, ok := stopWords[word]; !ok {
					tokens = append(tokens, word)
				}

				term.Reset()
			}
			continue
		}

    // convert to lowercase for case-insensitivity
		term.WriteRune(unicode.ToLower(r))
	}

	return tokens
}
```

The `lex` function removes a few stop words and returns all tokens in one style (lowercase) to support case-insensitive searches.

# Ranking

When you enter a search term, you don't want to go through every other inapt material on the internet that has that term in it. Instead, you want the most pertinent ones to what you want in very few clicks. Nobody likes the second page of a search results response. Because there are many matches related to your search query, a search engine needs to rank the results in order of importance to what you're looking for.

There are tons of ranking algorithms for documents out there, but we would be implementing a simple per-term ranking algorithm called **Term Frequency, Inverse Document Frequency (tf-IDF).**

According to [Wikipedia](https://en.wikipedia.org/wiki/Tf%E2%80%93idf), the Term Frequency, Inverse Document Frequency (tf-IDF) is a numerical statistic intended to reflect how important a word is to a document in a collection or corpus. In other words, an algorithm that helps us score words/terms in a document based on how frequently they appear in other documents.

Intuitively the algorithm works by:

- Giving the word/term a high score if it frequently occurs in a document (Term Frequency)
- But if the word/term appears in many documents, then it is not unique. Give the word a low score. (Inverse Document Frequency)

{{<rawhtml>}}<br/>{{</rawhtml>}}

```txt
tfIDF = Term Frequency (tf) x Inverse Document Frequency (IDF)
```

{{<rawhtml>}}<br/>{{</rawhtml>}}

**Term Frequency** measures how many times a word/term appears in a document blob. Then it is normalized by dividing the value by the total number of words in a blob text.

![Term Frequency (tf)](https://latex.codecogs.com/svg.image?\LARGE&space;tf_{t}=\frac{TC_{t}}{TTC})

where,

> **TC** - no of times the term `t` appears in the document \
> **TTC** - the total number of terms in the document

{{<rawhtml>}}<br/>{{</rawhtml>}}

**Inverse Document Frequency** is a weight that indicates how commonly a word is in use within our index. The more frequent its usage across documents, the lower its score. The lower the score, the less important the word becomes.

For any word/term `t` in the vocabulary,

![Inverse Document Frequency (IDF)](https://latex.codecogs.com/svg.image?\LARGE&space;IDF_{t}=\log_{10}\left[\frac{N}{DF_t}\right])

where,

> **N** - the total number of documents in the index \
> **DF** - the total number of documents with term `t`

**IDF** is typically used to boost the scores of words unique to a document with the hope that you surface high information words that characterize your document and suppress words that don't carry much weight in a document.

For example, the word "**the**" appears in almost all English texts and would thus have a very low IDF score as it carries very little 'topic' information. In contrast, if you take the word "**coffee**" while it is common, it is not used as widely as "**the**". Thus, "**coffee**" would have a higher IDF score than "**the**".

Extending `PostingNode` & `PostingList` to implement **tf-IDF** can be represented by:

```go
// PostingNode represents a document
type PostingNode struct {
  // document id
  docId int

  // term frequency
  tf float64

  // next node in the list
  next *PostingNode
}

// PostingList stores the posting list & data for a term
type PostingList struct {
  // total number of nodes
  n int32

  // inverse document frequency
  idf float64

  head *PostingNode
}
```

Bringing it all together:
