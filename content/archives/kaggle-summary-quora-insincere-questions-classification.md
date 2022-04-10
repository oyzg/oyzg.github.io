---
title: "[Kaggle Summary] Quora Insincere Questions Classification"
date: 2021-03-12T17:13:34+08:00
draft: false
math: true
categories: [Deep Learning, Natural Language Processing, Classification Problem]
series: [Kaggle Summary]
tags:
  [
    LSTM,
    kaggle,
    tensorflow,
    keras,
    imbalanced classification,
    class weight,
    bias initializer,
    tokenization,
    lemmatization,
    stemming,
    text vectorization,
    stop words,
    bag of words,
    TF-IDF,
    word2vec,
  ]
summary: "The summary of the notebook about Quora insincere questions classification in Kaggle. The main idea of the notebook is how to solve an imbalanced text classification problem with LSTM networks and Word2vec."
---

## Introduction

This article is a summary of the notebook about [Quora insincere questions classification](https://www.kaggle.com/c/quora-insincere-questions-classification) in Kaggle.

The link of kaggle notebook is below.

> https://www.kaggle.com/ouyangwenguang/quora-insincere-questions-classification

The main idea of the notebook is how to solve an imbalanced text classification problem with LSTM networks and Word2vec.

> **Important**: The complete code of examples below can be found in [Github](https://github.com/wgouyang/notebook/blob/master/kaggle-summary-quora-insincere-questions-classification.ipynb) or [Colab](https://colab.research.google.com/github/wgouyang/notebook/blob/main/kaggle-summary-quora-insincere-questions-classification.ipynb).

## Key Points

### Text Preprocessing

**`Tokenization`** is the process of tokenizing or splitting a string, text into a list of tokens. One can think of token as parts like a word is a token in a sentence, and a sentence is a token in a paragraph.

Examples of `tokenization` with NLTK:

```python
from nltk.tokenize import word_tokenize
from nltk.tokenize import sent_tokenize

corpus = '''Tokenization is the process of tokenizing or splitting a string, text into a list of tokens. One can think of token as parts like a word is a token in a sentence, and a sentence is a token in a paragraph.'''

print(sent_tokenize(corpus))
print(word_tokenize(corpus))
```

> output:

```log
['Tokenization is the process of tokenizing or splitting a string, text into a list of tokens.', 'One can think of token as parts like a word is a token in a sentence, and a sentence is a token in a paragraph.']
['Tokenization', 'is', 'the', 'process', 'of', 'tokenizing', 'or', 'splitting', 'a', 'string', ',', 'text', 'into', 'a', 'list', 'of', 'tokens', '.', 'One', 'can', 'think', 'of', 'token', 'as', 'parts', 'like', 'a', 'word', 'is', 'a', 'token', 'in', 'a', 'sentence', ',', 'and', 'a', 'sentence', 'is', 'a', 'token', 'in', 'a', 'paragraph', '.']
```

See also: https://www.nltk.org/api/nltk.tokenize.html#module-nltk.tokenize

**`Lemmatization`** is a normalization technique in the field of natural language processing. There have a related technique called `stemming`. In some languages, the word has difference forms in difference contexts. The goal of both `stemming` and `lemmatization` is to reduce various forms to a common base form.

Examples of `lemmatization` with NLTK:

```python
from nltk.stem import WordNetLemmatizer

corpus = ['rocks', 'gone', 'better']
lemmatizer = WordNetLemmatizer()

print([lemmatizer.lemmatize(w) for w in corpus])
```

> output:

```log
['rock', 'gone', 'better']
```

The result confused me. Why the output of `gone` and `better` isn't the base form of them? Because we don't provide the second parameter `parts-of-speech` to lemmatize. The parameter default value is `n` means a noun.

> `Lemmatization` needs the context to know what the `parts-of-speech` of the word is, a noun, verb or adjective so that can work well.

Usage:

```python
def lemmatize_sent(text):
    pos_dict = {'NN':'n', 'JJ':'a', 'VB':'v', 'RB':'r'}
    word_list = []
    for word, tag in pos_tag(word_tokenize(text)):
        pos = pos_dict[tag[0:2]] if tag[0:2] in pos_dict else 'n'
        word_list.append(lemmatizer.lemmatize(word, pos=pos))
    return word_list

sentence = 'He is walking to school'
lemmatization_words = [lemmatizer.lemmatize(w) for w in word_tokenize(sentence)]
print('lemmatize word by word: ', lemmatization_words)
print('lemmatize with context: ', lemmatize_sent(sentence))
```

> output:

```log
lemmatize word by word:  ['He', 'is', 'walking', 'to', 'school']
lemmatize with context:  ['He', 'be', 'walk', 'to', 'school']
```

**`Stemming`** shortens words with some rules and can be used without context, so it's more efficient than `Lemmatization`.

Examples of `Stemming` with NLTK:

```python
from nltk.stem import PorterStemmer

corpus = ['rocks', 'going', 'history']
stemmer = PorterStemmer()
print([stemmer.stem(w) for w in corpus])
```

> output:

```log
['rock', 'go', 'histori']
```

As you can see, the `stemming` outputs may not be actual words.

> Time-consuming Test:

```python
from nltk.corpus import gutenberg

@timing
def stemming(text):
    [stemmer.stem(w) for w in word_tokenize(sentence)]

@timing
def lemmatize(text):
    lemmatize_sent(text)

@timing
def lemmatize_without_context(text):
    [lemmatizer.lemmatize(w) for w in word_tokenize(sentence)]

book = gutenberg.raw("austen-sense.txt")

stemming(book)
lemmatize(book)
lemmatize_without_context(book)
```

> output:

```log
stemming                      : 0.22    ms
lemmatize                     : 5980.39 ms
lemmatize_without_context     : 0.17    ms
```

See also: https://www.nltk.org/api/nltk.stem.html#module-nltk.stem

**`Stop words`** usually refers to the most common words in a language, such as the, is, at, which and a in English. We remove the `stop words` before processing the text data because these words usually are no enough valuable.

Examples of `Stop words` with NLTK:

```python
from nltk.corpus import stopwords

corpus = ['I', 'am', 'a', 'boy']
print([w for w in corpus if w not in set(stopwords.words('english'))])
```

> output:

```log
['I', 'boy']
```

See also: https://www.nltk.org/book/ch02.html#stopwords_index_term

**`Text Vectorization`** is the process of converting text into a numerical vector that can be fed into a neural network or machine learning model. The most used methods to do that including `Bag of Words`, `TF-IDF` vectorization and `Word2vec`.

Examples of `Bag of Words` with scikit-learn:

```python
from sklearn.feature_extraction.text import CountVectorizer

vectorizer = CountVectorizer()
corpus = [
    'He is a teacher',
    'I am student',
    'She is also a student',
]
X = vectorizer.fit_transform(corpus)

print(vectorizer.get_feature_names())
print(X.toarray())
```

> output:

```log
['also', 'am', 'he', 'is', 'she', 'student', 'teacher']
[[0 0 1 1 0 0 1]
 [0 1 0 0 0 1 0]
 [1 0 0 1 1 1 0]]
```

The default configuration tokenizes the string by extracting words of at least 2 letters. so the word a has been discarded. The vector representation table of words is below.

```log
V(also)    : [1 0 0 0 0 0 0]
V(am)      : [0 1 0 0 0 0 0]
V(he)      : [0 0 1 0 0 0 0]
V(is)      : [0 0 0 1 0 0 0]
V(she)     : [0 0 0 0 1 0 0]
V(student) : [0 0 0 0 0 1 0]
V(teacher) : [0 0 0 0 0 0 1]

V(He is a teacher) = V(He) + V(is) + V(teacher) = [0 0 1 1 0 0 1]
...
```

TF means term-frequency.
$$\text{TF(t, d)}=\frac{\text{number of term t occurs in the document d}}{\text{number of terms in the document d}}$$

IDF means inverse document-frequency.  
$$\text{IDF(t)}=\log\frac{\text{number of documents in the corpus}}{\text{number of document where the term t occurs}}$$

TF-IDF means term-frequency times inverse document-frequency.

$$\text{TF-IDF(t, d)}=\text{TF(t, d)} \times \text{IDF(t)}$$

Examples of `TF-IDF` Verctorization with scikit-learn:

```python
from sklearn.feature_extraction.text import TfidfVectorizer

vectorizer = TfidfVectorizer()
corpus = [
    'He is a teacher',
    'I am student',
    'She is also a student',
]
X = vectorizer.fit_transform(corpus)

print(vectorizer.get_feature_names())
print(X.toarray())
```

> output:

```log
['also', 'am', 'he', 'is', 'she', 'student', 'teacher']
[[0.         0.         0.62276601 0.4736296  0.         0.         0.62276601]
 [0.         0.79596054 0.         0.         0.         0.60534851 0.        ]
 [0.5628291  0.         0.         0.42804604 0.5628291  0.42804604 0.        ]]
```

> Note: The calculational formula of TF-IDF has some changes in the TfidfVectorizer.

**`Word2vec`** is an algorithm that uses a shallow feedforward neural network to learn word embedding from a large corpus of text. There have two methods for producing a distributed representation of words.

- `Continuous Bag-of-Words` which was usually called `CBOW` predicts the current word from a window of surrounding context words. The order of context words does not influence prediction (bag-of-words assumption).
- `Continuous Skip-gram` uses the current word to predict the surrounding window of context words.

See also: https://www.tensorflow.org/tutorials/text/word2vec

Examples of `Word2vec` with gensim:

```python
import gensim.downloader
from gensim.models import Word2Vec

word2vec = gensim.downloader.load('word2vec-google-news-300')
print(word2vec.most_similar('car'))
print(word2vec.word_vec('car'))
```

> output:

```log
[('vehicle', 0.7821096181869507),
 ('cars', 0.7423830032348633),
 ('SUV', 0.7160962820053101),
 ('minivan', 0.6907036304473877),
 ('truck', 0.6735789775848389),
 ('Car', 0.6677608489990234),
 ('Ford_Focus', 0.667320191860199),
 ('Honda_Civic', 0.662684977054596),
 ('Jeep', 0.6511331796646118),
 ('pickup_truck', 0.64414381980896)]
 [ 0.13085938  0.00842285  0.03344727 -0.05883789  0.04003906 -0.14257812
  0.04931641 -0.16894531  0.20898438  0.11962891  0.18066406 -0.25
  ...
  0.04248047  0.12792969 -0.27539062  0.28515625 -0.04736328  0.06494141
 -0.11230469 -0.02575684 -0.04125977  0.22851562 -0.14941406 -0.15039062]
```

See also: https://radimrehurek.com/gensim/models/word2vec.html

### LSTM network

**`LSTM`** is an artificial recurrent neural network architecture. Its full name is long short-term memory, it is well-suited to classifying, processing and making predictions based on time series data.  
A common `LSTM` unit is composed of a cell, an input gate, an output gate and a forget gate. The cell remembers values over arbitrary time intervals and the three gates regulate the flow of information into and out of the cell.

See also: https://colah.github.io/posts/2015-08-Understanding-LSTMs/

The following is an explanation of some parameters that were being used in the model.

The `metrics` parameter is used to evaluate the model during training and testing. It uses `F1-score` as the final evaluation metric in this task,.
$$\text{F1-score}=2\cdot\frac{\text{precision}\cdot\text{recall}}{\text{precision}+\text{recall}}$$
`Precision` is the number of true positive results divided by the number of all positive results, including those not identified correctly.  
`Recall` is the number of true positive results divided by the number of all samples that should have been identified as positive.

See also: https://en.wikipedia.org/wiki/F-score

The `bias_initializer` parameter can influence the initial output of the layer. E.g. You have an imbalanced dataset of the ratio of positives to negatives is 1: 9. The model will predict 50% positive and 50% negative at initialization (PS: before training) if you don't set the `bias_initializer` parameter.
But if you have set the parameter properly, it will predict 10% positive and 90% negative at initialization, it is more reasonable. In addition, setting it correctly will speed up convergence.

The calculation of the `bias_initializer` parameter depends on what activation function the layer used. The correct bias to set can be derived from below if the activation function is `sigmoid`.
$$p_0=\frac{\text{positive}}{\text{positive}+\text{negative}}=\frac{1}{1+\exp(-b_0)}$$
then
$$b_0=-\log(\frac{1}{p_0}-1)=\log(\frac{\text{positive}}{\text{negative}})$$
See also: [Classification on imbalanced data](https://www.tensorflow.org/tutorials/structured_data/imbalanced_data#optional_set_the_correct_initial_bias)

You can refer to this [question](https://stackoverflow.com/questions/60307239/setting-bias-for-multiclass-classification-python-tensorflow-keras) if the activation function is `softmax`.

The `class weight` parameter is also important for a model during training if the dataset is imbalanced. Giving a heavier weight to an under-represented class will cause the model to pay more attention to it.

## Final Thoughts

When I finish the article, I learned more than when I begin to write. I found some approaches more convenient to preprocess text, such as `TextVectorization` in tensorflow. I hava a more in-depth understand of `Word2Vec`. I just knew it was a distributed representation of words. Now I know there have two methods to implement it, `CBOW` and `skip-gram`. The `skip-gram` model uses the technique called negative sampling to maximize the probability of predicting context words when given the target word.

## References

1. [Stemming and Lemmatization](https://nlp.stanford.edu/IR-book/html/htmledition/stemming-and-lemmatization-1.html)
2. [Basic NLP with NLTK](https://www.kaggle.com/alvations/basic-nlp-with-nltk#Stemming-and-Lemmatization)
3. [NLP | How tokenizing text, sentence, words works](https://www.geeksforgeeks.org/nlp-how-tokenizing-text-sentence-words-works/)
4. [Quick Introduction to Bag-of-Words (BoW) and TF-IDF for Creating Features from Text](https://www.analyticsvidhya.com/blog/2020/02/quick-introduction-bag-of-words-bow-tf-idf/)
5. [Word2Vec - Tensorflow](https://www.tensorflow.org/tutorials/text/word2vec)
6. [Long short-term memory - Wikipedia](https://en.wikipedia.org/wiki/Long_short-term_memory)
