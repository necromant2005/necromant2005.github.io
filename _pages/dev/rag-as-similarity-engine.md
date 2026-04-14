---
title: RAG As Similarity Engine
permalink: /dev/rag-as-similarity-engine/
---

# RAG As Similarity Engine

Most people talk about [RAG (Retrieval-Augmented Generation)](https://en.wikipedia.org/wiki/Retrieval-augmented_generation) like it is mainly for chatbots. On [hayovasweets.com/recipes](https://www.hayovasweets.com/recipes), I use it for something much simpler: finding similar recipes. For this kind of site, that feels like a much more normal use case. I just wanted a simple way to show related recipes in the same locale without building some bigger recommendation system.

## How Similarity Works

When a new recipe is added, the engine builds its embedding and compares it with the other recipe vectors by cosine similarity. That produces a shortlist of recipes that are close in ingredients, cuisine, preparation pattern, and overall cooking content. For a recipe site, that is honestly more useful than a chatbot in many cases because people are already browsing pages and just need good next suggestions.

So the flow is pretty simple: turn each recipe into one clean text document, ask OpenAI to convert that text into an embedding vector, store the vector in the index, and later compare saved vectors to find the closest recipes. OpenAI helps with the embedding part, but OpenAI alone does not rank your stored recipes for you. Once the vectors are saved, your own code still has to compare them and sort the best matches.

## How It Is Applied

The nice part is that the recipe itself becomes the source for similarity: title, ingredients, instructions, timing, tags, and other structured fields. That works much better than comparing pages by generic text or by tags alone, and it also keeps similar recipes in the language of the current page.

The embedding model configured in the project is `text-embedding-3-small`. After indexing, matching works from stored JSON data, so runtime stays simple.

## In Code

This is closer to the real indexing flow:

```ruby
require "openai"

def build_recipe_document(recipe)
  [
    "Title: #{recipe.title}",
    "Category: #{recipe.category}",
    "Cuisine: #{recipe.cuisine}",
    "Keywords: #{recipe.keywords.join(', ')}",
    "Ingredients:\n- #{recipe.ingredients.join("\n- ")}",
    "Instructions:\n" + recipe.instructions.each_with_index.map { |step, i| "#{i + 1}. #{step}" }.join("\n")
  ].join("\n\n")
end

def embed(text, api_key)
  client = OpenAI::Client.new(access_token: api_key)
  response = client.embeddings(
    parameters: {
      model: "text-embedding-3-small",
      input: text
    }
  )

  response.dig("data", 0, "embedding")
end

document = build_recipe_document(recipe)
embedding = embed(document, ENV.fetch("OPENAI_API_KEY"))

index_row = {
  id: "#{recipe.category_slug}/#{recipe.slug}",
  recipe: recipe.slug,
  category: recipe.category_slug,
  title: recipe.title,
  link: "/recipes/#{recipe.category_slug}/#{recipe.slug}",
  lang: recipe.lang,
  chunk: document,
  embedding: embedding
}
```

`build_recipe_document` matters because embeddings are only as good as the text you send into the model. If the input is clean and structured, the similarity result is usually much better too. `embed` is the OpenAI step that turns recipe text into a vector, and `index_row` is the persistence step that saves the vector so the site does not need to call the model again every time someone opens a recipe page.

That `index_row` ends up as JSON that looks roughly like this:

```json
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
```

And this is the basic similarity calculation:

```ruby
def cosine_similarity(left, right)
  dot = left.zip(right).sum { |a, b| a * b }
  left_size = Math.sqrt(left.sum { |value| value * value })
  right_size = Math.sqrt(right.sum { |value| value * value })

  return 0.0 if left_size.zero? || right_size.zero?

  dot / (left_size * right_size)
end
```

This part is still needed even when OpenAI is used. OpenAI gives you embeddings, but your code still has to decide which stored recipe is closest to which other recipe. `cosine_similarity` is the math that turns "two vectors" into "one similarity score."

Then matching recipes is just sorting by score:

```ruby
source_embedding = [0.12, -0.08, 0.33, 0.41]

recipes = [
  { slug: "breakfast-quesadilla", embedding: [0.10, -0.05, 0.31, 0.39] },
  { slug: "avocado-toast-with-egg", embedding: [0.09, -0.01, 0.22, 0.35] },
  { slug: "banana-pancakes", embedding: [0.02, 0.14, 0.05, 0.11] }
]

matches = recipes.map do |recipe|
  recipe.merge(score: cosine_similarity(source_embedding, recipe[:embedding]))
end

top_matches = matches.sort_by { |recipe| -recipe[:score] }.first(2)
```

That final step is what makes the feature usable. Without sorting, you just have a bunch of numbers. After sorting, you have a real list of the closest recipes to the current one.

## Limits

Right now, the recipe index stores one structured document per recipe. That is fine for this use case, but bigger or more detailed recipes might benefit from smaller chunks later. Retrieval quality also depends heavily on metadata quality, so if recipe content, tags, keywords, or translated snippets are inconsistent, semantic search becomes less reliable. So the hard part is not really "adding AI." The hard part is keeping the content clean and consistent.
