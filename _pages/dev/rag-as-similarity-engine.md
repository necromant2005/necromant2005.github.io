---
title: RAG As Similarity Engine
permalink: /dev/rag-as-similarity-engine/
---

# RAG As Similarity Engine

Most people talk about [RAG (Retrieval-Augmented Generation)](https://en.wikipedia.org/wiki/Retrieval-augmented_generation) as a chatbot pattern. For a recipes site like [hayovasweets.com/recipes](https://www.hayovasweets.com/recipes), a much simpler use case makes more sense: use the same idea to find similar recipes.

The goal is not to answer questions. The goal is to take one recipe and find other recipes that are close in ingredients, preparation style, and overall content.

## How The Flow Works

A general Python project for this usually does four things. First, it turns each recipe into one clean text document. Second, it sends those documents through a local embedding model built on PyTorch and Sentence Transformers. Third, it stores the resulting vectors in JSON. Fourth, it compares vectors with cosine similarity and keeps the closest matches.

That is enough to build a useful similarity engine. No chatbot is needed, and no large recommendation system is required either.

## Project Shape

A normal Python layout for this could look like this:

```text
src/
  recipe_rag/
    __init__.py
    documents.py
    embed.py
    similarity.py
    index.py
scripts/
  build_recipe_index.py
data/
  recipes/
    input/
      en.json
    output/
      en.json
requirements.txt
```

`documents.py` builds the text that will be embedded. `embed.py` loads the Sentence Transformers model and creates vectors. `similarity.py` compares vectors with PyTorch. `index.py` ties the flow together. A small script like `scripts/build_recipe_index.py` is enough to run the process from the command line.

## Environment

This kind of project usually just needs a normal virtual environment and a small requirements file. For example:

```text
sentence-transformers
torch
numpy
```

Then the setup is standard:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m recipe_rag.index --lang en
```

That command shape is much more natural for Python than application-style CLI naming.

## Building Documents

The first step is to turn one recipe into one clean text block. That part matters because the model only sees text. If the text is noisy, the embeddings are noisy too.

```python
def build_recipe_document(recipe: dict) -> str:
    sections = []

    if recipe.get("title"):
        sections.append(f"Title: {recipe['title']}")
    if recipe.get("category"):
        sections.append(f"Category: {recipe['category']}")
    if recipe.get("cuisine"):
        sections.append(f"Cuisine: {recipe['cuisine']}")
    if recipe.get("keywords"):
        sections.append(f"Keywords: {', '.join(recipe['keywords'])}")
    if recipe.get("ingredients"):
        sections.append("Ingredients:\n- " + "\n- ".join(recipe["ingredients"]))
    if recipe.get("instructions"):
        steps = "\n".join(
            f"{i + 1}. {step}" for i, step in enumerate(recipe["instructions"])
        )
        sections.append("Instructions:\n" + steps)

    return "\n\n".join(sections).strip()
```

This is where recipe text is actually used. Title, ingredients, instructions, timing, and other fields all end up in the final text document before embedding.

## Embedding With PyTorch

The next step is local embedding. A practical default is `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`, especially if recipes exist in multiple languages.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2")

texts = [doc["chunk"] for doc in documents]
embeddings = model.encode(
    texts,
    convert_to_numpy=True,
    normalize_embeddings=True,
    show_progress_bar=True,
)
```

Normalization matters because the next step is cosine similarity. Using normalized embeddings keeps that comparison clean and predictable.

## Stored Output

The embedded result can be stored as JSON and reused later. That keeps the runtime cheap because the model only needs to run when the index is rebuilt, for example when a new recipe is added.

```json
{
  "version": 1,
  "lang": "en",
  "embedding_provider": "sentence-transformers",
  "embedding_model": "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2",
  "documents": [
    {
      "id": "breakfast/huevos-rancheros",
      "recipe": "huevos-rancheros",
      "category": "breakfast",
      "title": "Huevos Rancheros",
      "link": "/recipes/breakfast/huevos-rancheros",
      "lang": "en",
      "chunk": "Title: Huevos Rancheros\n\nCategory: Breakfast\n\nCuisine: Mexican\n\nKeywords: mexican breakfast, eggs, tortillas, avocado\n\nIngredients:\n- 1 red bell pepper\n- 1 small onion\n- 300 g tomatoes\n- 4 eggs\n- 4 tortillas\n- 2 avocados\n\nInstructions:\n1. Dice tomatoes, onion, and pepper.\n2. Cook vegetables into a thick tomato sauce.\n3. Warm tortillas in a pan.\n4. Fry the eggs.\n5. Assemble with avocado, feta, and herbs.",
      "embedding": [0.021, -0.014, 0.057, -0.008]
    }
  ]
}
```

## Similarity Step

An embedding model gives you vectors, but the model does not rank stored recipes for you. The engine still needs a comparison step, and in a PyTorch project that can stay very small.

```python
import torch
import torch.nn.functional as F

source = torch.tensor(source_embedding, dtype=torch.float32)
others = torch.tensor(other_embeddings, dtype=torch.float32)

scores = F.cosine_similarity(others, source.unsqueeze(0), dim=1)
top_indices = torch.argsort(scores, descending=True)[:6]
```

That is the step that turns stored embeddings into actual related recipes.

## Ask As Natural-Language Search

The same index can also work like search. Instead of starting from an existing recipe, the engine starts from a user question such as "something with avocado and eggs" or "fast breakfast recipe." The flow is almost the same: turn the question into an embedding, compare that vector with the stored recipe vectors, and rank the nearest matches.

```python
query = "fast breakfast recipe with avocado and eggs"
query_embedding = model.encode(
    [query],
    convert_to_numpy=True,
    normalize_embeddings=True,
    show_progress_bar=False,
)[0]

query_tensor = torch.tensor(query_embedding, dtype=torch.float32)
scores = F.cosine_similarity(others, query_tensor.unsqueeze(0), dim=1)
top_indices = torch.argsort(scores, descending=True)[:6]
```

At that point, the engine already has a useful natural-language search result. If needed, those matched recipes can be passed to a second step that formats an answer, but retrieval itself is just embedding plus vector comparison.

## Why This Shape Works

This setup is simple for a reason. The content is already structured, Python handles the indexing flow cleanly, PyTorch handles the embedding work, and JSON is enough for storing the results. That is usually all you need when the job is “find similar recipes” rather than “build a general AI product.”
