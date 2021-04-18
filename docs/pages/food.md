---
sort: 5
---

# Food Recommendation

## Overview

The Food Recommender (FoodRec, in lieu of a catchy acronym) system is a prototype
for recommending healthy recipes to users. FoodRec makes use of RDF data about
recipes and ingredients from the [FoodKG](https://foodkg.github.io/). 
The aim of FoodRec is to provide another example of how FREx can be used to
create pipelines to choose good recommendations, and we create some pipeline
stages related to healthy eating guidelines.

In addition to FoodKG, we use some embeddings for recipes that were produced by
some related work, [RECIPTOR](https://dl.acm.org/doi/10.1145/3394486.3403223).
Embeddings of recipes produced by RECIPTOR are used in FoodRec to check for 
similarity between recipes, and we use this similarity in generating recipe
candidates to recommend. 

The implementation of FoodRec is viewable on [github](https://github.com/CognitiveHorizons/RPI-HEALS-FoodKG-Semantic-Substitutions)
 - todo this isn't a public repo, moving it soon  

## Data and Resources

The main resource we use is the [FoodKG](https://foodkg.github.io/). FoodKG provides
us with a knowledge graph of recipes, which include information about the ingredients
used in the recipe. Ingredients in turn have links to information about their
nutritional content, based on data from the [USDA](https://fdc.nal.usda.gov/ndb/foods),
and their classification in [FoodOn](https://foodon.org/).

## Recommendation Processes

A key pipeline in the FoodRec application is the RecommendRecipesPipeline. 
This pipeline consumes the current user as context and produces recommendations
for recipes. The stages in this pipeline score recipes based on how similar they
are to the user's favorite recipes (based on cosine similarity of their
RECIPTOR embeddings), filter recipes that use ingredients that are prohibited
by the user, and apply additional stages based on whether or not certain 
healthy-eating guidelines are applicable to the user.   

``` python
class RecommendRecipesPipeline(Pipeline):
    
    def __init__(...):
        ...

        Pipeline.__init__(self, stages=(
                    SimilarToFavoritesRecipeCandidateGenerator(
                        recipe_embedding_service=self.res,
                        food_kg_query_service=self.food_kg,
                    ),
                    ContainsAnyProhibitedIngredientFilter(
                        filter_explanation=Explanation(
                            explanation_string="This recipe does not contain any ingredients that are prohibited by you."
                        )
                    ),
                    ApplyGuidelinesToRecipesPipeline(
                        guideline_kg=guideline_kg
                    ),
                    RecipeCaloriesScorer(
                        scoring_explanation=Explanation(
                            explanation_string="Scoring based on calories, this is mostly a placeholder to break ties."
                        )
                    ),
                    CandidateRanker()
            ),
        )
``` 

The `ApplyGuidelinesToRecipesPipeline` is a pipeline that contains health guidelines
as stages. For each stage, the system checks whether the user meets the conditions
specified by the Guideline - if they do, that stage is applied to the recipe candidate
as it passes through the pipeline. 
In the current iteration of FoodRec, these guidelines are implemented manually. 
In a more developed application, it would be preferable to be able to automatically
parse some authoritative source of information pertaining to health and eating guidelines
to produce such stages automatically, but since this wouldn't be a trivial endeavor
we leave it for future work. 

In FoodRec, we also make use of the constraint solver. Unlike ESCoRe, the constraint
solver in FoodRec is made to solve a much more simple constraints to form meal plans
from the recommended recipes. The `RecommendMealPlanPipeline` first uses the
`RecommendRecipesPipeline` to get a list of candidate recipes that are good
for the user. It then passes these recipes along to the `MealPlanCandidateGenerator`,
which serves to generate a new MealPlan candidate based on the highly ranked recipes coming
from the previous pipeline stage.

``` python
class RecommendMealPlanPipeline(Pipeline):

    def __init__(...):
        ...
        Pipeline.__init__(self, stages=(
                RecommendRecipesPipeline(
                    recipe_embedding_service=recipe_embedding_service,
                    ...
                ),
                MealPlanCandidateGenerator(
                    number_of_days=number_of_days, meals_per_day=meals_per_day
                ),
            ),
        )
```

## Example Output

output examples
