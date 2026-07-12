
1. LLM - A neural network trained to predict the next token.
2. Tokenization -  Text -> Tokens -> Numbers ->Neural Network
3. Token IDs - Once split every token gets an ID.


Why IDs are not enough ?
```
Dog -> 542
Cat -> 911
Car -> 87

Can the neural network infer Dog and Cat are similar?
No.

542 and 911 are just arbitrary integers.
There is no mathematical relationship.
```

Embeddings
	Embedding captures semantic meaning.
	Not spelling.
	Not grammar.
	Meaning.
```
Instead of representing Dog as  542
we represent it as vector 

[
0.12,
-0.55,
0.87,
...
768 numbers
]

This list is called an embedding.
```



# Cosine Similarity

Instead of Euclidean distance LLMs usually use  Cosine Similarity.

Why?

Because direction matters more than magnitude.

Two vectors pointing in nearly the same direction represent similar meanings, even if one is longer than the other.

For vectors **A** and **B**:

cosine similarity=A⋅B∥A∥∥B∥\text{cosine similarity} = \frac{A \cdot B}{\|A\| \|B\|}cosine similarity=∥A∥∥B∥A⋅B​

The result ranges from **-1 to 1**:

- **1** → identical direction (very similar)
- **0** → unrelated
- **-1** → opposite direction

Example:

```
Dog -> [1, 2]
Cat -> [2, 4]
```

The second vector is just twice the first, so their cosine similarity is **1.0**—they're treated as semantically identical despite different magnitudes.


# How Embeddings Are Learned

Initially, embeddings are random.

```
Dog

[0.45,
-0.21,
1.01]

Cat 

[-2.5,
0.1,
0.6]
```

No structure exists.

During training:
1. The model predicts the next token.
2. It computes the prediction error (loss).
3. Using back-propagation, it adjusts all learnable parameters, including the embedding vectors.
4. After billions of updates, semantically related tokens naturally move closer together because that helps the model make better predictions.
