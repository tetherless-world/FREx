---
sort: 2
---

# FREx Features

## Modeling

FREx is developed around the use of RDF data. Since raw RDf is not particularly
well suited to perform typical object oriented programming, we will want some 
service present to convert RDF into Python classes. How exactly you want to do
that is slightly out of scope of this documentation, although you can see some 
examples in the [university course recommendation](/FREx/pages/escore.html) and 
[healthy food recommendation](/FREx/pages/food.html) implementations.

To convert RDF into Python classes, we of course need to define the Python classes.
In FREx, the base class for any entities that have a URI is the [`DomainObject`](https://solashirai.github.io/FREx/build/html/frex.models.html#module-frex.models.domain_object).
When modeling custom classes for your domain, they should subclass the DomainObject.

 ```python
from dataclasses import dataclass
from dataclasses_json import dataclass_json
from frex.models import DomainObject

@dataclass_json
@dataclass(frozen=True)
class MyItem(DomainObject):
    name: str
    cost: float
    color: str
``` 

```note
In FREx, we implement DomainObjects using the [dataclass](https://docs.python.org/3/library/dataclasses.html) 
decorator. This decorator lets you skip over a lot of boilerplate code, like defining
`__init__`, `__eq__`, `__hash__` and so on, for classes that mainly serve to
just store data.  
```

Another important base class to keep in mind for FREx is the `Candidate`. While
a class that extends the DomainObject reflects entities/items related to the recommendation
process, Candidates encapsulate information about candidate items that might be
recommended to a user. They also store information about scores and explanations
that have been applied to the candidate as it pasaes through a recommendation pipeline. 

You can extend the Candidate class if desired, to specify
more accurate type hints, but this is not strictly necessary.

```python
from frex.models import Candidate
from my_app.models import MyItem
from dataclasses import dataclass
from dataclasses_json import dataclass_json

@dataclass_json
@dataclass
class MyCandidate(Candidate):
    domain_object: MyItem
```

## Recommendation Pipeline

FREx is most effective in supporting the development of systems
that produce recommendations through a series of steps to __generate__, __filter__,
and __score__ candidate items. The basic principal behind these steps is fairly
straight forward. A system can start off by generating a set of initial candidates,
not necessarily candidates that are perfect as final answers but rather a quick set
of plausible items. These candidates then can be filtered, for example by
removing candidates that are __definitely__ not suitable to the user. 
Candidates then can be scored based on some strategies that are relevant
to your application, and based on these scores you can re-rank the candidates
to choose your final recommendations.

FREx controls this overall process using the Pipeline class, and the various
the stages that candidates pass through are implemented as PipelineStages.

### Pipeline

The [`Pipeline`](https://solashirai.github.io/FREx/build/html/frex.pipelines.html) is
a callable class. Its initialization arguments include the stages that are involved in
the pipeline. These "stages" can either be PipelineStage classes or other
Pipeline classes, which can let you reuse full Pipelines within Pipelines.

Calling the Pipeline will run the underlying pipeline stages, passing
along the relevant arguments of `context` (which can be anything - depends on what context
is relevant in your application) and `candidates`. Since you generally will
start running your pipeline without a set of candidates to start with,
you typically would call a pipeline just with the context argument.

You can make your own Pipelines by subclassing FREx's pipeline class, which can 
be convenient if you want to set up your stages.
```python
from frex.pipelines import Pipeline
from frex.pipeline_stages.scorers import CandidateRanker 
from my_app.pipeline_stages import MyGenerator, MyFilterer, MyScorer

class MyPipe(Pipeline):
    def __init__(self, *):
        Pipeline.__init__(
            self, 
            stages=(
                MyGenerator(...),
                MyFilterer(...),
                MyScorer(...),
                CandidateRanker()))

...

my_pipe = MyPipe()
my_context = ...
top_5_results = list(my_pipe(context=my_context))[:5]
``` 

### Pipeline Stages

[`PipelineStage`](https://solashirai.github.io/FREx/build/html/frex.pipeline_stages.html#frex.pipeline_stages.pipeline_stage.PipelineStage)s
are also callable classes, and serve as the stages that take place in
a Pipeline. The three most basic stages that are useful in a typical Pipeline have
base classes implemented in FREx to support easy expansion.

PipelineStages consume and yield Python generators, which are generating
candidate items for the next stage to consume and process.

#### CandidateGenerator

`CandidateGenerator` is a stage for generating new candidates to pass through
the pipeline. New CandidateGenerator subclasses can be minimally created by just
defining a `generate` function, but you probably will also need to specify
some data source from which you're generating candidates as well.

```python
from frex.pipeline_stages.candidate_generators import CandidateGenerator
from my_app.models import MyCandidate 

class MyGenerator(CandidateGenerator):
    def generate(self, *):
        # for the sake of brevity, lets pretend that we have some method
        # here to retrieve all items, and have that serve as our example generator
        all_items = ...
        # yield new MyCandidate objects to start moving through the pipeline
        for item in all_items:
            yield MyCandidate(...)
``` 

#### Candidate Filterer

`CandidateFilterer` is a stage for filtering unsuitable candidates. From
the base class, custom filterers can be minimally constructed by defining the
`filter` function. Note that `filter(candidate)` returning `True` indicates
that the candidate **is** filtered from the pipeline and is removed from consideration
for your recommendation.

```python
from frex.pipeline_stages.filters import CandidateFilterer 
from my_app.models import MyCandidate

class MyFilterer(CandidateFilterer):
    def filter(self, *, candidate: MyCandidate):
        # filter out candidate items that are blue
        return MyCandidate.domain_object.color == 'blue'
``` 

#### Candidate Scorer

A `CandidateScorer` stage scores the candidate based on some scoring function
that you define. Similarly to the filterer, a new scorer can minimally
be created by just defining the `score` function. 

```python
from frex.pipeline_stages.scorers import CandidateScorer 
from my_app.models import MyCandidate

class MyScorer(CandidateScorer):
    def score(self, *, candidate: MyCandidate):
        # score a candidate based on its price
        return MyCandidate.domain_object.cost**2
``` 

FREx also has a `CandidateRanker` class, which can be inserted into a pipeline
to aggregate the scores applied so far to the candidates and order them. This can
be used as the last stage in a pipeline to act as the final tallying and re-ranking
of candidate items to choose the final recommendation(s).

## Explanations

As Candidates are generated and pass through each stage of a pipeline in FREx,
we want to be able to provide explanations about what was done to Candidates.
Currently, we do this using an `Explanation` class, which simply consists of a 
string (i.e., the explanation text). For each pipeline stage, when yielding
a Candidate to the next stage, an explanation and score is attached to update
the Candidate. Explanation are defined by the developer and attached to stages
at initialization.

More complicated explanations can be created by extending the Explanation class.
The Explanation class is __very__ simple, since our main motivation for it was
to provide a more structured representation to capture the explanations rather
than passing around magic strings every which way.

 ```python
from frex.pipelines import Pipeline
from frex.pipeline_stages.scorers import CandidateRanker 
from frex.models import Explanation
from my_app.pipeline_stages import MyGenerator, MyFilterer, MyScorer

example_pipe = Pipeline(stages=(
                MyGenerator(...),
                MyFilterer(
                    filter_explanation=Explanation(
                        explanation_string="This item isn't blue."
                    ) 
                ),
                MyScorer(
                    scorer_explanatoin=Explanation(
                        explanation_string="This item is expensive."
                    ) 
                ),
                CandidateRanker()))
``` 

## Outputs

After running a Pipeline to generate recommendations, the output results
will be Candidate objects that are populated with relevant information 
that was passed in by each of the stages the Candidate passed through. 
This includes the context involved in the recommendation pipeline,
the actual object that is being recommended, scores applied to the candidate,
and explanations supplied by each stage.

``` 
output_results = [
    MyCandidate(
        #the context used to run the pipeline, whatever you want to define
        #context to be in your application. This might be as simple as
        #just pointint to the user for which this recommendation was produced
        context=MyUser(...),
        #the item being recommended
        domain_object=MyItem(...),
        #scores applied to the candidate by each pipeline stage
        applied_scores=[0.5, 0, 3.9],
        #explanations applied by each pipeline stage
        applied_explanations=[Explanation(...), ...]
    ),
    ...]
``` 
