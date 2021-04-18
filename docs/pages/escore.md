---
sort: 4
---

# Course Recommendation

## Overview

The Explainable Semantic Course Recommender (ESCoRe) system is a prototype
system developed using FREx for recommending university courses to students.
More specifically, it's made for students attending Rensselaer Polytechnic
Institute (RPI). ESCoRe uses domain objects modeled by the 
[Course Recommender Ontology](https://rpi-ontology-engineering.netlify.app/oe2020/course-recommender/) (CRO),
which was designed and developed for this purpose. More detailed documentation
about the CRO, which describes more details about the conceptual model and
relevant considerations, can be found on the CRO's website.

ESCoRe's main functionality is to provide recommendations for courses to students.
Additionally, we implemented a constraint solving system, using Google's OR-Tools,
to enable ESCoRe to generate full plans for student courses. These full plans
(or plan-of-study) are generated such that the courses meet all of the student's
graduation requirements. 

The implementation of ESCoRe is viewable on [github](https://github.com/solashirai/ExplainableCourseRecommender/).

## Data and Resources

Besides the [CRO](https://rpi-ontology-engineering.netlify.app/oe2020/course-recommender/ontology),
our system uses RPI's public academic catalog to collect information about courses.

Currently, graduation requirements have been added manually based on CRO's modeling
of graduation requirements, since there is no well-structured resource that's
publicly available about all of the graduation requirements at RPI. 
Due to this limitation, in ESCoRe graduation requirements are only fully implemented
for Computer Science majors. 

## Recommendation Processes

ESCoRe's recommendation pipelines mostly focus on checking for __correctness__
in recommendations - i.e., ensuring that you have fulfilled pre-requisites to
take a course, or checking that a course is actually offered at a certain semester.
The following pipeline shows an example of the stages involved in a pipeline for
recommending courses to a student for a given semester.

``` python
class RecommendCoursesForSemesterPipeline(Pipeline):
    def __init__(self, *, course_query_service: CourseQueryService):
        self.cqs = course_query_service
        Pipeline.__init__(
            self,
            stages=(
                CourseSectionsInSemesterCandidateGenerator(
                    course_query_service=self.cqs
                ),
                UndergradCourseFilter(
                    filter_explanation=Explanation(
                        explanation_string="This is an undergraduate-level course."
                    )
                ),
                PrerequisiteUnfulfilledFilter(
                    filter_explanation=Explanation(
                        explanation_string="You have fulfilled the required prerequisites to take this course."
                    )
                ),
                CanFulfillPOSRequirementFilter(
                    filter_explanation=Explanation(
                        explanation_string="This course can fulfill some requirement towards your degree."
                    )
                ),
                RecommendedPrereqScorer(
                    scoring_explanation=Explanation(
                        explanation_string="You have completed some of the recommended prerequisites for this course."
                    )
                ),
                TopicOfInterestScorer(
                    scoring_explanation=Explanation(
                        explanation_string="This course covers topics that you have indicated as being interested in."
                    ),
                    course_query_service=self.cqs,
                ),
                CandidateRanker(),
            ),
        )
``` 

For generating recommendations for a full plan-of-study, the pipelines are fairly similar.
After generating all courses and scoring them, the courses are passed along to
ESCoRe's `PlanOfStudyRecommenderService`. This service takes the graduation
requirements related to the current student, formats them into a state
that is usable by our constraint solving implementation, and passes the
scored candidate courses in. The constraint solving system essentially then
serves to choose the set of courses that maximizes the score of the final plan
while also fulfilling all constraints (i.e., graduation requirements, 
coure pre-requisite requirements, and course offering times).

## Example Output

For simple course recommendations, we can look at a comparison of some of the
top Computer Science courses recommended to two fake students that have
interests in different topics.

| Ranking | StudentA - Machine Learning | StudentB - Ontology Engineering |
| ---- | ---- | ---- |
| 1 | Machine Learning from Data | Ontologies |  
| 2 | Human Computer Interaction | Data Mining |  
| 3 | Data Analytics | Computational Social Processes |  
| 4 | Data Mining | Frontiers of Network Science |  
| 5 | Intelligent Agents | Database Systems |

In the full plan-of-study generated for students, we also are able to 
check that some constraints (like prerequisites) are correctly addressed.
The table below shows a small snippet of some courses selected for 
StudentA, who has an interest in machine learning.

| Semester | Course Name | Pre-requisite Courses |
| ---- | ---- | ---- |
| Fall 2019 | Computer Science I |   |
| ---- | ---- | ---- |
| ... | ... | ... |
| ---- | ---- | ---- |
| Spring 2020 | Calculus I |  |
| ---- | ---- | ---- |
| Spring 2020 | Data Structures | Computer Science I |
| ---- | ---- | ---- |
| Fall 2020 | Foundations of Computer Science | Data Structures, Calculus I |
| ---- | ---- | ---- |
| Fall 2020 | Calculus II | Calculus I |
| ---- | ---- | ---- |
| ... | ... | ... |
| ---- | ---- | ---- |
| Spring 2021 | Introduction to Algorithms | Data Structures, Calculus I, Foundations of Computer Science |
| ---- | ---- | ---- |
| ... | ... | ... |
| ---- | ---- | ---- |
| Fall 2022 | Machine Learning from Data | Introduction to Algorithms|
| ---- | ---- | ---- |
| ... | ... | ... |
| ---- | ---- | ---- |

There are a couple of major limitations to the current prototype.
First, due to how course information is displayed in RPI's course catalog,
not all courses could be accurately parsed to determine their pre-requisite
courses (partially due to ambiguity, sometimes due to poor or
inconsistent formatting). This causes some errors with course selection.

Another major limitation for generating a full plan-of-study is the system's
runtime. Our current prototype implementation takes __all__ courses offered
at RPI and plugs them into a constraint solver. Since performance wasn't a
big focus of this prototype example, the solver is not very well optimized.
As a result, it can sometimes take a fairly large amount of time to generate
a plan for a single student. Runtimes were not always consistent, but
generating a plan for a student with no completed courses (i.e., generating
a full plan for 4 years / 8 semesters of university) could take around 30 minutes.
