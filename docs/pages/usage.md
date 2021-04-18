---
sort: 3
---

# Using FREx

## Suggested Project Structure

While FREx doesn't require any particular project structure, we believe
that using the following structure can provide a good foundation for
developing recommender systems using FREx. This structure is also used
in our two example applications. 

```
my_app/
|-- models/
|   |-- my_item.py
|   |-- my_user.py
|
|-- pipelines/
|   |-- my_recommender_pipeline.py
|
|-- pipeline_stages/
|   |-- my_item_generator.py
|   |-- my_item_scorer.py
|   |-- my_item_filter.py
|
|-- services/
|   |-- item_query_service.py
|   |-- user_query_service.py
``` 

In this example, we'll assume that our
application has two relevant things to model in the domain - users and items -
and some recommender pipeline to go with them.

The FREx Pipeline and PipelineStages used in this application should be
working with MyItem or MyUser objects, rather than having to pass around
raw RDF through the application. Here, services (like the ItemQueryService
in this example) should serve the purpose of querying a knowledge graph or
RDF database and then convert the query results into Python objects.
This will help follow good encapsulation practices.

FREx also has some simple support for this structure of using services
to query RDF data, whether it be loading some local files or querying
a remote datasource using SPARQL. These can be found in the
[`frex.stores`](https://solashirai.github.io/FREx/build/html/frex.stores.html)
subpackage. Examples of using these stores can be seen in our example applications,
e.g., in the `food_rec/services/food/graph_food_kg_query_service.py`.

