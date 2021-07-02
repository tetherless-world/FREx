---
sort: 1
---

# Getting Started

## Overview

The purpose of FREx is to provide a structure in which you can develop a
recommender systems, especially those that use RDF data and use rule-driven
steps to determine what to recommend.
FREx uses a pipeline system to process candidate items to recommend, and
it also attaches explanations about each step that a candidate went through
in the recommendation pipeline.
Therefore, outputs of a recommendation pipeline implemented using FREx will
have explanations about the processes it went through as well as the
context in which the candidates were produced attached as a key piece of the
final output.

FREx is **NOT** a framework that takes in your data and spits out a ranking of 
items to recommend. Rather, it is a framework in which you would incorporate
other recommender algorithms to score or generate candidates. 
FREx also is **NOT** a tool that automatically generates explanations. The
explanations are created by the developer - FREx helps to provide a structure
in which such explanations can be passed along and incorporated through the
pipeline.   

## Installation

To use FREx, you should be using Python version 3.8 or higher. This 
supports the use of [dataclass decorators](https://docs.python.org/3/library/dataclasses.html)
as well as some quality-of-life improvements surrounding type hints. 

You can use FREx by installing it via pip
```bash
$ pip install git+https://github.com/solashirai/FREx@master#egg=frex
```

You will now be able to use FREx in your project like any other Python package, with `import frex`.

```warning
One of FREx's requirements, [ortools](https://pypi.org/project/ortools/), may require you to 
be using a 64-bit installation of Python. If you are seeing an error related to ortools 
while trying to pip install FREx, consider checking which Python installation you are using.  
```
