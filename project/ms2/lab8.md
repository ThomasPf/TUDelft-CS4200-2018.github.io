---
layout: page
title: "Lab 8: Data-Flow Analysis"
excerpt: "Lab 8: Data-Flow Analysis"
tags: ["project"]
context: project
subcontext: ms2
---

{% include _toc.html %}

This lab is under construction. Proceed at own risk.
{: .notice .notice-warning}

In this lab, you define a simple data-flow analysis for MiniJava in FlowSpec. The concepts you are going to use in FlowSpec are described in the following paper:

- Jeff Smits, Eelco Visser. FlowSpec: declarative dataflow analysis specification. In Benoît Combemale, Marjan Mernik, Bernhard Rumpe, editors, Proceedings of the 10th ACM SIGPLAN International Conference on Software Language Engineering, SLE 2017, Vancouver, BC, Canada, October 23-24, 2017. pages 221-231, ACM, 2017.

## Overview

### Objectives

Specify a reaching definitions analysis for MiniJava in FlowSpec. The specification should include:

1. The control-flow of methods (including the main method) in MiniJava, including
   - blocks,
   - if statements,
   - while loops,
   - print statements,
   - assignments,
   - and array assignments.
2. Reaching definitions analysis, taking into account
   - parameters,
   - variables,
   - fields,
   - assignments,
   - and array assignments.
3. Messages on use of uninitialized variables, in particular
   - errors on uninitialized variables.

### Submission

You need to submit your MiniJava project with a merge request against branch `assignment-8-submission` on GitLab.
The [Git documentation](/documentation/git.html#submitting-an-assignment) explains how to file such a request.

The deadline for submission is December 1, 2018.
{: .notice .notice-warning}

### Grading

You can earn up to 100 points for the correctness of your data-flow analysis.Therefore, we run several test cases against your implementation. You earn points, when your implementation passes test cases. The total number of points depends on how many test cases you pass in each of the following groups:

- control flow (35 points)
- reaching definitions analysis (35 points)
- error messages (10 points)
- challenge (20 points)

### Early Feedback

We provide early feedback for your language implementation. This feedback gives you an indication
which parts of your name and type analysis might still be wrong. It includes a summary on how many
tests you pass and how many points you earn by passing them. You have 3 early feedback attempts.

## Detailed Instructions

### Preliminaries

#### GitLab Repository

We provide you with a template for this assignment in the `assignment-4-template` branch. See the
[Git documentation](/documentation/git.html#template) on how to check out this branch.

The template contains 4 Maven projects:

* `minijava`: an initial MiniJava project that covers concrete and abstract syntax, desugarings, and name and type analysis,
* `minijava.example`: an empty project for your example MiniJava programs,

#### Importing projects into Eclipse

The projects from the template have to be imported into Eclipse:

1. Start Eclipse.
2. Select your workspace.
3. Import the projects into your workspace:
    1. Right-click into the Package Explorer.
    2. Select Import... from the context menu.
    3. Choose Maven -> Existing Maven Projects from the list.
    4. Press next.
    5. Select the root of your Git repository.
    6. Select all projects.
    7. Make sure that Copy projects into workspace is **off**.
    8. Press finish

If you get an error about `Resource '/.org.eclipse.jdt.core.external.folders' already exists`, you can safely ignore that.

### Control-flow

Control-flow is specified as control-flow rules in `.flo` files. FlowSpec files must go in the `trans/analysis` directory. The module name at the top of the file should match the filename relative to `trans`. For example, the file `trans/analysis/control.flow` starts as:

```
module analysis/control
```

#### Control-flow rules

The rules match on an AST constructor, and define the control-flow of that construct in terms of nodes, control-flow of the subtrees, and the surrounding control-flow graph. 

```
  control-flow rules // Lists

  Cons(h, t) = entry -> h -> t -> exit
  Nil() = entry -> exit
```

The surrounding control-flow are the `entry` and `exit` nodes. You can use matched subtrees from the AST as placeholders for their control-flow. You can also create a control-flow graph node associated with an AST node using `node matchedVar`. Most commonly you want to add a node to the control-flow graph corresponding to the entire matched tree on the left. You can use `this` for that situation:

```
UnExp(_, e) = entry -> e -> this -> exit
```

At the leaves of the AST you will then see patterns such as `entry -> this -> exit`. Since this is so common, there is another shortcut for this. This is the `node` rule, where you only match the AST pattern, and do not need a right-hand side:

```
node True()
```

The right-hand side of a control-flow rule really specifies a list of edges, separated by commas. Because most control-flow is simply a chain, you can chain multiple edges together. 

There can only be **one** rule that matches on a certain constructor. 
{: .notice .notice-warning}

#### Control-flow root

A control-flow graph is usually rooted at the procedure -- or in MiniJava's case method -- level. A special root rule specifies this, and uses an explicit `start` and `end` node instead of the usual contextual `entry` and `exit`. 

```
root Method(_, _, params, vars, body) = start -> ... -> end
```

### Data-flow

A data-flow analysis in FlowSpec is built on a data-flow property. This property is defined in a `properties` section where it is given a name and a lattice. The lattice is a combination of a type and some operations that automatically deal with splitting and merging the data when the control-flow splits and merges. 

```
properties

  live: MaySet(name)
```

#### Rules

A data-flow analysis is either a forward or a backward analysis. A forward analysis specifies what information is available at the `start` of the control-flow, a backward analysis specifies what information is available at the `end` of the control-flow. 

```
property rules

  live(Method(_, _, _, _, _).end) = {}
```

When the direction of a data-flow analysis is established with such a rule, FlowSpec will automatically propagate information along that direction. Therefore you only need to write rules for control-flow nodes that have a special meaning in your analysis. You again use AST patterns, the ones associated with the control-flow graph nodes, to match on the left side. But now you also give a name to the neighboring control-flow graph node using an edge pattern:

```
  live(VarRef(n) -> next) = live(next) \/ { Var{n} }
```

In this example analysis, the backward direction is visible in that we name the `next` node on the right-hand side of the arrow, and match the left-hand side of the arrow. We define the liveness information of the matched `VarRef` here. Therefore by making it equal to the information of `next` (and more), we propagate information backward through the graph. 

#### Expressions

[FlowSpec expressions](http://www.metaborg.org/en/latest/source/langdev/meta/lang/flowspec/reference.html#expressions) that you are likely to need are set operations. These are union (`\/`), intersection (`/\`), minus (`\`), containment (`in`), set construction (`{ elements, go, here }`), and comprehension (`{ new | old <- set, conditions }`). 

### Reaching definitions

You will implement a classic data-flow analysis called reaching definitions. This analysis gives information about where a variable was last assignment a value. Note that a variable could be last assigned in multiple places at once if you check this information after a merge in control-flow. Therefore your analysis should work on a set of pairs. The first part of the pair is the name, the second part of the pair is a position where the assignment took place. 

For the reaching definitions data-flow analysis, we require you to name your analysis `reaching`. This is used in the grading for the lab. 
{: .notice .notice-warning}

A name is constructed with NaBL2 syntax (`namespace { occurrence }`), although we do not differentiate between declarations and references in FlowSpec. A position of an AST node can be requested with the `position` function. 

You should also take into account that local variables are not initialized when first declared. Your reaching definitions analysis should support this, as we will use this analysis to display errors on uses of uninitialized variables. 

Consider what should happen to the data at a merge of control-flow. This indicates whether you should use the `MaySet` or the `MustSet` lattice. 

### Error messages

Once you have finished your data-flow analysis, you can use it to generate error messages. [NaBL2 allows you to hook into the analysis and create extra errors, warnings and notes](http://www.metaborg.org/en/latest/source/langdev/meta/lang/nabl2/stratego-api.html#custom-analysis). In this case we add FlowSpec analysis based on the name analysis results, and then we query the analysis results to construct error messages:

```
  nabl2-custom-analysis-unit-hook:
    (_, ast, _) -> ast

  nabl2-custom-analysis-final-hook(|a):
    (_, _, asts) -> (errors, warnings, notes, a2)
    with <nabl2-custom-analysis-info(|"Running FlowSpec for more errors")> a
       ; a2 := <flowspec-analyze(|a)> asts
       ; errors   := <error-uninitialized(|a2)> asts
       ; warnings := []
       ; notes    := []
```

To get the ast in the final phase, we pass it along in the unit phase. We use `flowspec-analyze(|a)` to execute the FlowSpec analysis on top of the NaBL2 analysis `a`, and receive the combined scope graph, control-flow graph and data-flow properties in `a2`. We pass this information to strategy `error-uninitialized`. This strategy should give back a list of pairs, the first being the ast node to put the error on, the second a string to display. 

Implement `error-uninitialized` so that error messages are displayed on every use of a variable or parameter where that variable or parameter _may_ be uninitialized. 

### Challenge

For the challenge we want a slightly different behavior from your implementation than Java. In Java a local variable _must_ be initialized before it can be used, anything else is an error. For this challenge, give a warning instead, iff the local variable is _possibly_ uninitialized at a use site. 
