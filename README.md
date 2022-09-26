# The Cocktail Lounge

This project is designed to demonstrate the use of RPA, graphs, and AI to generate novel cocktail recipes. It is a proof of concept only, with examples and sample data to replicate or change the data flow. There may well be different or better ways to achieve each of the included systems and actions.

The overall flow of the solution is:
1. UiPath to extract and normalize the data
2. Neo4j to curate and analyze the data
3. GPT-3 to tune AI based on the data and use random data inputs to feed results


## /UiPath
Extract cocktail recipe data from Kindred Cocktails (https://www.kindredcocktails.com) and normalize the data for downstream use. The project contains 2 workflows, which can be run independently or scripted to run in sequence.

* Main.xaml: Iteratively scrape raw text from each recipe page on Kindred's website. Output is an XLSX file containing cocktail names, URLs, and raw recipes.
* GenerateImportCSVs.xaml: Manipulate raw data to create node properties, normalize values, and format special characters. Output is two CSV files, one representing Cocktail nodes and one representing Ingredient nodes.


## /Neo4j
Ingest cocktail recipe data in a graph database to visualize, analyze, and utlimately help tune an AI model. The key output from this graph is a list of "Affinity Sets", a.k.a. a set of three ingredients where each ingredient occurs frequently with each of the other two. The idea is that these affinity sets provide a good basis from which to generate new cocktails, since the ingredients seemingly pair well with each other.

### Build the Graph
To replicate the sample database, consider the following steps and Cypher queries:

1. Create a Neo4j database and add the CSV files to the project's `/import` folder.
2. Import Cocktail Nodes to the database
```
LOAD CSV WITH HEADERS FROM 'file:///cocktails.csv' AS row 
CREATE (:Cocktail {name: row.Name, url: row.URL})
```
3. Import Ingredient Nodes to the database
```
LOAD CSV WITH HEADERS FROM 'file:///ingredients.csv' AS row
MERGE (ingredient:Ingredient {name: row.Name})
```
4. Import PART_OF relationships from Ingredients to Cocktails
```
LOAD CSV WITH HEADERS FROM 'file:///ingredients.csv' AS row
MATCH (i:Ingredient {name: row.Name}), (c:Cocktail {name: row.Cocktail})
CREATE (i)-[:PART_OF {quantity: row.Quantity, unit: row.Unit}]->(c)
```
5. Create affinity relationships between ingredients that share 5+ recipes
```
MATCH (a:Ingredient)--(c:Cocktail)--(b:Ingredient)
WHERE id(a)>id(b)
WITH a, b, COUNT(a) as Freq
FOREACH (i IN CASE WHEN Freq >= 5 THEN [1] ELSE [] END |
MERGE (a)-[:AFFINITY {frequency: Freq}]-(b))
```
6. Generate a random affinity 'set', which can be used to feed the AI model
```
MATCH (a:Ingredient)-[r:AFFINITY]-(b:Ingredient)-[s:AFFINITY]-(c:Ingredient)-[t:AFFINITY]-(d:Ingredient)
WHERE a=d and id(b)>id(c)
WITH a, b, c, (r.frequency+s.frequency+t.frequency) AS totalFreq, rand() as num
RETURN a.name, b.name, c.name, totalFreq
ORDER BY num DESC, totalFreq DESC
LIMIT 1
```

### Other Queries
Here are some other useful queries:

* Show a cocktail recipe
```
MATCH (:Cocktail {name: 'Auld Lang'})-[r]-(i)
RETURN i.name, r.quantity, r.unit
```
* Return the ingredients occurring in the same cocktails as a given ingredient
```
MATCH (a:Ingredient)--(c:Cocktail)--(i:Ingredient {name: 'Aperol'})
RETURN a.name, c.name
```
* Show all the cocktail recipes associated with an affinity set
```
MATCH (c:Cocktail)-[:PART_OF]-(:Ingredient {name: 'Bitters'}),
(c)-[:PART_OF]-(:Ingredient {name: 'Falernum'}),
(c)-[:PART_OF]-(:Ingredient {name: 'Absinthe'})
RETURN c.name
```
* Export training data for GPT-3 (Note: this requires some massaging to prepare the prompt-completion formatting)
```
MATCH (c:Cocktail)-[r]-(i)
RETURN c.name as Name, reverse(collect((r.quantity+' '+r.unit+' '+i.name))) as Recipe, reverse(collect(i.name)) as Ingredients
LIMIT 3000
```

## /OpenAI
Use the GPT-3 text generator as a basis for tuning against cocktail recipe-specific data. THe base GPT-3 model can generate reasonable cocktail recipes, however they tend to be very mundane or totally incomrehensible. By tuning the model on ingredient inputs and recipe outputs, all results conform to the target format and are generally much more useful.

Fine-tuning and using a custom GPT-3 model requires the following steps, all of which entail calls to the OpenAI API (I used Postman):

1. Upload a tuning file, in JSONL format, containing sample prompts and completions. The sample file `cocktail_training.jsonl` contains 3,000 records pulled from Neo4j and formatted to include a standard prompt.
```
POST https://api.openai.com/v1/files

{
  "file": "file-name-here",
  "purpose": "fine-tune"
}
```

2. Run fine-tune training by selecting a base model, the file to ingest, and any other parameters.
```
POST https://api.openai.com/v1/fine-tunes

{
  "training_file": "file-name-here",
  "model": "curie",
  "suffix": "cocktail-recipes"
}
```

3. Execute against the new model by providing prompts, tweaking other parameters to further tune results.
```
POST https://api.openai.com/v1/completions

{
  "model": "curie:ft-personal:cocktail-recipes",
  "prompt": "Suggest a new cocktail recipe with Egg White, Triple Sec, Tequila, and other ingredients.\n\n###\n\n",
  "max_tokens": 125,
  "temperature": 0.75,
  "n": 3,
  "stop": "END",
  "presence_penalty": 0.2,
  "frequency_penalty": 0.2
}
```