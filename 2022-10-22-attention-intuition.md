---
title: "An Intuition for Attention"
date: 2022-10-22
description: "Deriving the equation for scaled dot product attention."
---

The transformer neural network architecture is the secret sauce behind LLMs (large language models) like ChatGPT and the models that power [cohere.ai](https://cohere.ai). The main feature of the transformer is a mechanism called _attention_. While attention can come in many different forms, the attention mechanism used in the transformer architecture is defined as:
$$\text{attention}(Q, K, V) = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V$$
This version of attention (known as scaled dot product attention), was first proposed in the original [transformer paper]( https://arxiv.org/pdf/1706.03762.pdf), is still widely used by most transformer implementations. In this post, we'll build an intuition for the above equation by deriving it from the ground up.

To start, let's take a look at the problem attention aims to solve, the key-value lookup.

## Key-Value Lookups
---
A key-value (kv) lookup involves three components:
1. A list of $n_k$ **keys**
2. A list of $n_k$ **values** (that map 1-to-1 with the keys, forming key-value pairs)
3. A **query**, for which we want to _match_ with the keys and get some value based on the match

You're probably familiar with this concept as a dictionary or hash map:

```python
>>> d = {
>>>     "racket": 10,
>>>     "ball": 5,
>>>     "tree": 2,
>>> }
>>> d.keys()
['racket', 'ball', 'tree']
>>> d.values()
[10, 5, 2]
>>> query = "racket"
>>> d[query]
10
```

Dictionaries let us perform lookups based on an _exact_ string match.

What if instead we wanted to do a lookup based on the _meaning_ of a word?

## Key-Value Lookups based on Meaning
---
Say we wanted to look up the word "tennis" in our previous example, how do we choose which "key" is the best match?

It's obviously not "tree", but both "racket" and "ball" seem appropriate matches. It's hard to choose one or the other, tennis feels more like a combination of racket and ball rather than a strict match for either.

So, let's not choose. Instead we'll do exactly that, take a combination of racket and ball. For example, say we assign a 60% meaning based match for racket, a 40% match for ball, and 0% match for tree, our output value would be the weighted sum of the values:

```python
>>> query = "tennis"
>>> d = {"racket": 10, "ball": 5, "tree": 2}
>>> 0.6 * d["racket"] + 0.4 * d["ball"] + 0.0 * d["tree"]
8
```

Essentially, we are determining how much **attention** our query should be paying to each key-value pair based on _meaning_. The amount of "attention" is represented as a decimal percentage, called an **attention score**, which we use to take the weighted sum. Mathematically, we can define our output as:
$$
\sum_{i} \alpha_iv_i
$$
where $\alpha_i$ is our attention score for the $i$th kv pair and $v_i$ is the $i$th value. Remember, the attention scores are decimal percentages, that is they must be between 0 and 1 inclusive ($0 \leq \alpha_i \leq 1$) and their sum must be 1 ($\sum_i a_i = 1$).

Okay, but where did we get these attention scores from? In our example, I just kind of chose them based on what I _felt_. While I think I did a pretty good job, this approach doesn't seem sustainable (unless you can find a way to make a copy of me inside your computer).

Instead, let's take a look at how **word vectors** may help solve our problem of determining attention scores.

## Word Vectors and Similarity
---
Image we represent a word with a vector of numbers. Ideally, this vector should in some way capture the _meaning_ of the word it represents. For example, imagine we have the following word vectors (visualized in 2D space):

![](https://i.imgur.com/VDnSf7P.png)

You can see that words that are "similar" are clustered together. Fruits are clustered at the top right, vegetables are clustered at the top left, and furniture is clustered at the bottom. In fact, you can even see that the vegetables and fruit clusters are closer to each other than they are to the furniture cluster, since they are more closely related things.

You can even imagine doing arithmetic on word vectors. For example, given the words "king", "queen", "man", and "woman" and their respective vector representations $\boldsymbol{v}_{\text{king}}, \boldsymbol{v}_{\text{queen}}, \boldsymbol{v}_{\text{man}}, \boldsymbol{v}_{\text{women}}$, we can imagine that:
$$\boldsymbol{v}_{\text{queen}} - \boldsymbol{v}_{\text{woman}} + \boldsymbol{v}_{\text{man}} \sim \boldsymbol{v}_{\text{king}}$$
That is, the vector for "queen" minus "woman" plus "man" should result in a vector that is _similar_ to the vector for "king".

But what does it exactly mean for two vectors to be _similar_? In the fruits/vegetables example, similarity meant how far apart they were in vector space (aka their [euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance)).

However there are many different ways to [measure the similarity between two vectors](https://towardsdatascience.com/9-distance-measures-in-data-science-918109d069fa), each with their own advantages and disadvantages. Possibly the simplest way measure of similarity is the dot product:
$$\boldsymbol{v} \cdot \boldsymbol{w} = \sum_{i}v_i w_i$$
[3blue1brown has a great video on the intuition behind dot product](https://www.youtube.com/watch?v=LyGKycYT2v0), but for our purposes all we need to know is:
* If two vectors are pointing in the same direction, the dot product will be > 0 (i.e. similar)
* If they are pointing in opposing directions, the dot product will be < 0 (i.e. disimilar)
* If they are exactly perpendicular, the dot product will be 0 (i.e. neutral)

Using this information, we can define a simple heuristic to determine the similarity between two word vectors: The greater the dot product, the more similar two words are in _meaning_.[^magnitude]

Okay cool, but where do these word vectors actually come from? In the context of neural networks, they usually come from some kind of learned embedding or latent representation. That is, initially the word vectors are just random numbers, but as the neural network is trained, their values are adjusted to become better and better representations for words. 

## Attention Scores using the Dot Product
---
If we treat our query and keys as word vectors instead of strings ($\boldsymbol{q} = \boldsymbol{v}_{\text{tennis}}$ and $\boldsymbol{k} = [\boldsymbol{v}_{\text{racket}} \ \boldsymbol{v}_{\text{ball}} \ \boldsymbol{v}_{\text{tree}}]$, all with dimensionality $d_k$), we can compute the similarity between our query and the $i$th key as a dot product:
$$
x_i = \boldsymbol{q} \cdot \boldsymbol{k}_i
$$
We can compute the dot product for all $n_k$ keys with:

$$
\boldsymbol{x} = \boldsymbol{q}{K}^T
$$
where $\boldsymbol{x}$ is our vector of dot products $\boldsymbol{x} = [x_1, x_2, \ldots, x_{n_k - 1}, x_{n_k}]$ and $K$ is a row-wise matrix of our key vectors (i.e. our key vectors stacked on-top of each-other to form a $n_k$ by $d_k$ matrix such that $k_i$ is the $i$th row of $K$). If you're having trouble understanding this, see [^matmul].

Recall that our attention scores need to be decimal percentages (between 0 and 1 and sum to 1). Our dot product values on the other hand, can be any real number (i.e. between $-\infty$ and $\infty$). To transform our dot product values to decimal percentages, we'll use the [softmax function](https://en.wikipedia.org/wiki/Softmax_function):
$$
\text{softmax}(\boldsymbol{x})_i = \frac{e^{x_i}}{\sum_j e^{x_j}}
$$

```python
>>> import numpy as np
>>> def softmax(x):
>>>     # assumes x is a vector
>>>     return np.exp(x) / np.sum(np.exp(x))
>>>
>>> softmax(np.array([4.0, -1.0, 2.1])) 
[0.8648, 0.0058, 0.1294]
```
Notice:
* ✅ Each number is between 0 and 1
* ✅ The numbers sum to 1
* ✅ The larger valued inputs get more "weight"
* ✅ The sorted order is preserved (i.e. the 4.0 is still the largest after softmax, and -1.0 is still the lowest), this is because softmax is [monotonic](https://en.wikipedia.org/wiki/Monotonic_function)

These are all the properties we needed for our attention scores, giving us the definition:
$$
\alpha_i = \text{softmax}(\boldsymbol{x})_i = \text{softmax}(\boldsymbol{q}K^T)_i
$$
Plugging this into our weighted sum we get [^values]:
$$
\begin{align}
\sum_{i}\alpha_iv_i
= & \sum_i \text{softmax}(\boldsymbol{x})_iv_i\\
= & \sum_i \text{softmax}(\boldsymbol{q}K^T)_iv_i\\
= &\ \text{softmax}(\boldsymbol{q}K^T)\boldsymbol{v}
\end{align}
$$
And that's it, we have a full working definition for attention:
$$
\text{attention}(\boldsymbol{q}, K, \boldsymbol{v}) = \text{softmax}(\boldsymbol{q}K^T)\boldsymbol{v}
$$
In code:

```python
import numpy as np

def get_word_vector(word, d_k=8):
    """Hypothetical mapping that returns a word vector of size
    d_k for the given word. For demonstrative purposes, we initialize
    this vector randomly, but in practice this would come from a learned
    embedding or some kind of latent representation."""
    return np.random.normal(size=(d_k,))

def softmax(x):
    # assumes x is a vector
    return np.exp(x) / np.sum(np.exp(x))

def attention(q, K, v):
    # assumes q is a vector of shape (d_k)
    # assumes K is a matrix of shape (n_k, d_k)
    # assumes v is a vector of shape (n_k)
    return softmax(q @ K.T) @ v

def kv_lookup(query, keys, values):
    return attention(
        q = get_word_vector(query),
        K = np.array([get_word_vector(key) for key in keys]),
        v = values,
    )

# returns some float number
print(kv_lookup("tennis", ["racket", "ball", "tree"], [10, 5, 2]))
```

## Scaled Dot Product Attention
---
In principle, the attention equation we derived in the last section is complete. However, we'll need to make a couple of changes to match the version in [Attention is All You Need](https://arxiv.org/pdf/1706.03762.pdf).

#### Values as Vectors
The values associated with each key need not be a singular number, they can be vectors of any size $d_v$. For example with $d_v = 4$, you might have:

```python
d = {
    "racket": [0.9, 0.2, -0.5, 1.0]
    "ball": [1.2, 2.0, 0.1, 0.2]
    "tree": [-1.2, -2.0, 1.0, -0.2]
}
```

In this case, the attention score gets multiplied to each number in the vector (scalar-vector multiplication).

To adjust for this change in our equation, instead of multiply our attention scores by a vector $v$ we multiply it by the row-wise matrix of our value vectors $V$ (similar to how we stacked our keys to form $K$):
$$
\text{attention}(\boldsymbol{q}, K, V) = \text{softmax}(\boldsymbol{q}K^T)V
$$
Of course, our output is no longer a scalar, instead it would be a vector of dimensionality $d_v$.

#### Scaling
The dot product between our query and keys can get really large in magnitude if $d_k$ is large. This makes the output of softmax more _extreme_. For example, `softmax([3, 2, 1]) = [0.665, 0.244, 0.090]`, but with larger values (say we multiply our inputs by 10) `softmax([30, 20, 10]) = [9.99954600e-01, 4.53978686e-05, 2.06106005e-09]`. When training a neural network, this would mean the gradients would become really small which is undesirable. As a solution, we scale our pre-softmax scores by $\frac{1}{\sqrt(d_k)}$:

$$
\text{attention}(\boldsymbol{q}, K, V) = \text{softmax}(\frac{\boldsymbol{q}K^T}{\sqrt{d_k}})V
$$

#### Multiple Queries
In practice, we often want to perform multiple lookups for $n_q$ different queries rather than just a single query. Of course, we could always do this one at a time, plugging each query into the above equation. However, if we stack of query vectors row-wise as a matrix $Q$ (in the same way we did for $K$ and $V$), we can compute our output as a $n_q$ by $d_v$ matrix where row $i$ is the output vector for the attention on the $i$th query:
$$
\text{attention}(Q, K, V) = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V
$$
that is, $\text{attention}(Q, K, V)_i = \text{attention}(q_i, K, V)$. On a CPU/GPU, this makes computation faster than if we ran attention for each query sequentially (say, in a for loop).

Note, our input to softmax becomes a matrix instead of a vector. When we write softmax here, we mean that we are taking the softmax along each row independently, as if we were doing things sequentially.

#### Result
With that, we have our final equation for scaled dot product attention as it's written in the original paper:
$$
\text{attention}(Q, K, V) = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V
$$In code:

```python
import numpy as np

def softmax(x):
    # assumes x is a matrix and we want to take the softmax along each row
    # (which is achieved using axis=-1 and keepdims=True)
    return np.exp(x) / np.sum(np.exp(x), axis=-1, keepdims=True)

def attention(Q, K, V):
    # assumes Q is a matrix of shape (n_q, d_k)
    # assumes K is a matrix of shape (n_k, d_k)
    # assumes v is a matrix of shape (n_k, d_v)
    # output is a matrix of shape (n_q, d_v)
    d_k = K.shape[-1]
    return softmax(Q @ K.T / np.sqrt(d_k)) @ V
```

[^mathnote]: We use the [standard math notation](https://www.deeplearningbook.org/contents/notation.html) defined in the [Deep Learning](https://www.deeplearningbook.org) book, that is: **scalars** are unbolded lowercase ($a$), **vectors** are bolded lowercase ($\boldsymbol{a}$), **matrices** are unbolded uppercase ($A$)
[^magnitude]: You'll note that the magnitude of the vectors have an influence on the output of dot product. For example, given 3 vectors, $a=[1, 1, 1]$, $b=[1000, 0, 0]$, and $c=[2, 2, 2]$, our dot product heuristic would tell us that becuase $a \cdot b > a \cdot c$  that $a$ is more similar to $c$ than $a$ is to $b$. This doesn't seem right, since $b$ and $a$ are pointing in the exact same direction, while $c$ and $a$ are not. [Cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity) accounts for this normalizing the vectors to unit vectors before taking the dot product, essentially ignoring the magnitudes and only caring about the direction. So why don't we take the cosine similarity? In a deep learning setting, the magnitude of a vector might actually contain information we care about (and we shouldn't get rid of it). Also, if we regularize our networks properly, outlier examples like the above should not occur.
[^values]: In the last step, we pack our values into a vector $\boldsymbol{v} = [v_1, v_2, ..., v_{n_k -1}, v_{n_k}]$, which allows us to get rid of the summation notation in favor of a dot product.
[^matmul]: Basically, instead of computing each dot product separately:
$$
\begin{align}
x_1 = & \ \boldsymbol{q} \cdot \boldsymbol{k}_1 = [2, 1, 3] \cdot [-1, 2, -1] = -3\\
x_2 = & \ \boldsymbol{q} \cdot \boldsymbol{k}_2 = [2, 1, 3] \cdot [1.5, 0, -1] = 0\\
x_3 = & \ \boldsymbol{q} \cdot \boldsymbol{k}_3 = [2, 1, 3] \cdot [4, -2, -1] = 3
\end{align}
$$
You compute it all at once:
$$
\begin{align}
\boldsymbol{x} & = \boldsymbol{q}{K}^T \\
& = \begin{bmatrix}2 & 1 & 3\end{bmatrix}\begin{bmatrix}-1 & 2 & -1\\1.5 & 0 & -1\\4 & -2 & -1\end{bmatrix}^T\\
& = \begin{bmatrix}2 & 1 & 3\end{bmatrix}\begin{bmatrix}-1 & 1.5 & 4\\2 & 0 & -2\\-1 & -1 & -1\end{bmatrix}\\
& = [-3, 0, 3]\\
& = [x_1, x_2, x_3]
\end{align}
$$
[^french]: I'm using "Je vous aime" instead of "Je t'aime" for simplicity so I can treat everything as separate words.