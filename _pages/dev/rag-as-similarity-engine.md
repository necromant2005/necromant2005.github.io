---
title: RAG As Similarity Engine
permalink: /dev/rag-as-similarity-engine/
---

# RAG As Similarity Engine

Most people talk about RAG(Retrieval-Augmented Generation) like it is mainly for chatbots. On [hayovasweets.com/recipes](https://www.hayovasweets.com/recipes), I use it for something much simpler: finding similar recipes. For this kind of site, that feels like a much more normal use case. I just wanted a simple way to show related recipes in the same locale without building some bigger recommendation system.

## How Similarity Works

When the site needs related recipes, it takes the current recipe, looks at its embedding, compares it with the other recipe vectors, and ranks the matches by cosine similarity. That gives a shortlist of recipes that are close in ingredients, cuisine, preparation pattern, and overall cooking content. For a recipe site, that is honestly more useful than a chatbot in many cases because people are already browsing pages and just need good next suggestions.

## How It Is Applied

The nice part is that the recipe itself becomes the source for similarity: title, ingredients, instructions, timing, tags, and other structured fields. That works much better than comparing pages by generic text or by tags alone, and it also keeps similar recipes in the language of the current page.

The embedding model configured in the project is `text-embedding-3-small`. After indexing, matching works from stored JSON data, so runtime stays simple.

## Sample Files

A setup like this could look like this:

```text
data/
  rag/
    recipes/
      index.en.json
      index.uk.json
  storage/
    recipes/
      breakfast/
        huevos-rancheros/
          snippet/
            en.json
            uk.json
```

Example of a localized recipe snippet file:

```json
{
  "title": "Huevos Rancheros",
  "category": "Breakfast",
  "cuisine": "Mexican",
  "keywords": [
    "mexican breakfast",
    "eggs",
    "tortillas",
    "avocado"
  ],
  "preparation_time": "PT15M",
  "cooking_time": "PT20M",
  "total_time": "PT35M",
  "servings": "4 servings",
  "nutrition": "420",
  "ingredients": [
    "1 red bell pepper",
    "1 small onion",
    "300 g tomatoes",
    "4 eggs",
    "4 tortillas",
    "2 avocados"
  ],
  "instructions": [
    "Dice tomatoes, onion, and pepper.",
    "Cook vegetables into a thick tomato sauce.",
    "Warm tortillas in a pan.",
    "Fry the eggs.",
    "Assemble with avocado, feta, and herbs."
  ]
}
```

Example of a generated RAG index file:

```json
{
  "version": 1,
  "lang": "en",
  "embedding_model": "text-embedding-3-small",
  "generated_at": "2026-03-30T12:55:05Z",
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

Example of a similar-recipes result:

```json
{
  "source_recipe": "huevos-rancheros",
  "top_matches": [
    {
      "recipe": "breakfast-quesadilla",
      "title": "Breakfast Quesadilla",
      "score": 0.913,
      "link": "/recipes/breakfast/breakfast-quesadilla"
    },
    {
      "recipe": "avocado-toast-with-egg",
      "title": "Avocado Toast With Egg",
      "score": 0.887,
      "link": "/recipes/breakfast/avocado-toast-with-egg"
    }
  ]
}
```

## Why This Works Better Than Generic AI

This works well mostly because the task is narrow and the input is clean. The system compares real recipe content instead of trying to produce broad answers from messy website text. The whole thing is also cheap to run because the model is only used during indexing, so once a recipe is added the stored vectors can be reused and ongoing similarity checks are almost free.

## Limits

Right now, the recipe index stores one structured document per recipe. That is fine for this use case, but bigger or more detailed recipes might benefit from smaller chunks later. Retrieval quality also depends heavily on metadata quality, so if recipe content, tags, keywords, or translated snippets are inconsistent, semantic search becomes less reliable. So the hard part is not really "adding AI." The hard part is keeping the content clean and consistent.
