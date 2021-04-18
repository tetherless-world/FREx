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

The implementation of FoodRec is viewable on [github](https://github.com/solashirai/FoodRec)

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

In our implemented example, we use a relatively small subset of FoodKG -
only 5,000 recipes compared to the roughly 1 million in the full dataset -
since the runtime of our pipelines and querying can become long when dealing
with huge knowledge graphs. The recommendations and similarity of recipe therefore
probably aren't too great, but the purpose of this prototype is moreso to view
how we can use FREx to develop an app and how it can show explanations.

The json below shows an example of a meal plan output for a test user, providing
2 recipes each day for 3 days. The explanations for each recipe correspond to
the stages that the recipe candidate passed through in the recommender pipeline.

``` json
mealplan_days:
    0:
        day_explanation:
            explanation_string: "This ia a set of recommended recipes to eat for this day, based on suggesting recipes that you are likely to like in general."
        meals:	
            0:	{…}
            1:	
                recipe_name: "Pizza with Pepperoni and Veggies"
                calories(kcal): 499.722919
                carbohydrates(g): 48.121708
                sodium(mg): 1599.402352
                explanation:
                    0:
                        explanation_string: "This recipe had a similarity score of 0.9967120851070816 to one of your favorite recipes, Open-Face Portabella Sandwiches."
                    1:	    
                        explanation_string: "This recipe does not contain any ingredients that are prohibited by you."
                    2:	    
                        explanation_string: "Adheres to guideline: As for the general population, people with diabetes should limit sodium consumption to <2,300 mg/day."
                    3:	
                        explanation_string: "Adheres to guideline: 1,500–1,800 kcal/day for men, adjusted for the individuals baseline body weight"
                ingredients:	
                    0: "Italian dressing"
                    1: "pepperoni"
                    2: "black olives"
                    3: "fresh thyme"
                    4: "red onions"
                    5: "fresh mushrooms"
                    6: "ready - to - use baked pizza crust"
                    7: "mozzarella cheese"
                    8: "cherry tomatoes"
    1:
        day_explanation: {…}
        meals: […]
    2: {…}
overall_explanation:   	
    explanation_string: "This ia a meal plan that was generated for 3 days of meals, eating 2 meals each day."
```
