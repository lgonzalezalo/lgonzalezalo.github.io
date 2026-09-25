---
layout: default
title: Revisiting a 2010 T-SQL Genetic Algorithm
description: A technical retrospective of a genetic-algorithm learning project written in 2010 and revisited in 2026.
---

# Revisiting a 2010 T-SQL Genetic Algorithm

> A technical retrospective of a genetic-algorithm learning project I wrote in 2010, published on my former Microsoft SQL Server blog, **sqlast.blogspot.com**, and revisited sixteen years later.

## TL;DR

In 2010, as part of my final project for UNED’s postgraduate programme **[Aprendizaje Estadístico y Data Mining (Plan 09)](https://formacionpermanente.uned.es/tp_actividad/actividad/aprendizaje-estadistico-y-data-mining-plan-09)**, I wrote a small genetic algorithm in T-SQL. Its purpose was educational: I wanted to understand genetic-algorithm mechanics by implementing binary encoding, fitness evaluation, selection, crossover, and mutation in a language I already knew well.

I later published the script on my SQL Server blog, sqlast.blogspot.com. It encodes two continuous variables as a 33-bit binary chromosome and maximizes a nonlinear function using two-elite selection, one-point crossover, and mutation.

Sixteen years later, I revisited it with a different question: **how has this code aged?** I used an AI assistant as an initial review tool to surface questions about representation, diversity, mutation semantics, numerical scaling, reproducibility, and observability. I then checked those questions against the original code and current SQL Server practice.

The conclusion is balanced: the original script remains a useful teaching artifact, but it is not a production optimization pattern. This repository preserves the historical version and includes a minimally revised version that makes its assumptions and behavior easier to inspect.

> **Scope.** This is a historical and educational exercise. It is not a recommendation to use SQL Server as the primary runtime for production evolutionary optimization.

## Contents

- [Why I wrote this code](#why-i-wrote-this-code)
- [Repository files](#repository-files)
- [What this is—and is not](#what-this-isand-is-not)
- [The optimization problem](#the-optimization-problem)
- [The chromosome: 33 bits for two variables](#the-chromosome-33-bits-for-two-variables)
- [From bits to real values](#from-bits-to-real-values)
- [Fitness evaluation](#fitness-evaluation)
- [Selection, crossover, and mutation](#selection-crossover-and-mutation)
- [What the original teaches—and where it breaks down](#what-the-original-teachesand-where-it-breaks-down)
- [Minimal 2026 revision](#minimal-2026-revision)
- [Why this is not production architecture](#why-this-is-not-production-architecture)
- [Conclusion](#conclusion)
- [Credits and provenance](#credits-and-provenance)

## Why I wrote this code

I wrote the original script in 2010 as part of the final project for the UNED postgraduate programme **[Aprendizaje Estadístico y Data Mining (Plan 09)](https://formacionpermanente.uned.es/tp_actividad/actividad/aprendizaje-estadistico-y-data-mining-plan-09)**.

At that point, I was exploring statistical learning, data mining, and heuristic search methods. My aim was not to build a production optimizer inside SQL Server. I wanted to understand the mechanics of a genetic algorithm by implementing its essential components in a language I already used and understood well: T-SQL.

The exercise made every design decision tangible:

- How should a candidate solution be encoded?
- How can a binary chromosome be decoded into bounded real-valued variables?
- How should fitness be calculated and stored?
- How can selection, crossover, and mutation be expressed with SQL Server strings, variables, table structures, and loops?
- What happens when simplicity of implementation is prioritized over population diversity and algorithmic sophistication?

I later published the script on my SQL Server blog because it was a compact and reproducible way to show that T-SQL could be used to explore algorithmic ideas beyond conventional data access, reporting, and stored-procedure work.

Sixteen years later, I came across the script again and wanted to examine it through a 2026 lens. The code still works as an understandable example, but it also reflects the trade-offs of its original purpose: simplicity over performance, extreme elitism over diversity, and a compact demonstration over experimental rigor.

I used an AI assistant as a starting point for the review and to generate questions about the implementation. The final analysis in this repository is based on checking those questions against the source code, the behavior of the algorithm, and current SQL Server practice. The goal was not to judge a 2010 learning exercise by 2026 production standards. It was to understand what it still teaches and what I would design differently today.

## Repository files

The repository contains two runnable SQL scripts and this article:

| File | Purpose |
|---|---|
| [Annotated historical script](./scripts/historical-sqlast-2010.sql) | Preserves the algorithmic structure of my 2010 post, with explanatory comments |
| [Minimally revised 2026 script](./scripts/minimally-revised-2026.sql) | Corrects numerical and selection details, makes mutation explicit, and adds observability |
| [Repository README](./README.md) | Quick-start instructions and project overview |
| [Provenance and attribution notice](./NOTICE.md) | Attribution and licensing scope |

## What this is—and is not

The historical script is a compact implementation of a classical genetic algorithm in T-SQL. It maintains a population of binary chromosomes, decodes them into candidate values, evaluates fitness, chooses parents, creates offspring through crossover and mutation, and repeats this cycle across generations.

It is useful because it maps concepts that are often presented in pseudocode to concrete SQL constructs:

| Genetic-algorithm concept | T-SQL representation |
|---|---|
| Individual | One row in a results table |
| Population | Rows belonging to the same generation |
| Genotype | A `DNA` binary string |
| Phenotype | Decoded `x1` and `x2` values |
| Fitness | The objective-function value |
| Selection | Ranking candidates by fitness |
| Crossover | String slicing with `LEFT()` and `RIGHT()` |
| Mutation | Replacing or flipping bits with `STUFF()` |

It is **not** a modern production optimizer. The implementation is procedural, evaluates candidates one by one, uses a scalar base-conversion UDF, and deliberately emphasizes simplicity over search quality, throughput, or sophisticated diversity preservation.

## The optimization problem

The script maximizes:

\[
f(x_1,x_2)=21.5+x_1\sin(4\pi x_1)+x_2\sin(20\pi x_2)
\]

subject to:

\[
-3.0\le x_1\le12.1
\]

\[
4.1\le x_2\le5.8
\]

The sinusoidal terms create several local maxima. That makes the function a reasonable illustration of a search problem in which an optimizer can explore different regions instead of following only one local direction.

The scale of the task should remain clear: this is a two-variable objective. A genetic algorithm is useful here as an instructional device, not because it is necessarily the simplest or most efficient practical method. In a real numerical workflow, a grid search, a global optimizer, differential evolution, CMA-ES, or a gradient-based method—when applicable—would typically be considered first.

## The chromosome: 33 bits for two variables

The original script stores each candidate in a table variable with fields equivalent to:

```sql
DECLARE @results table
(
    sequence_id int identity,
    GenerationNumber int,
    DNA varchar(33),
    x1 float,
    x2 float,
    fitness float
);
```

The chromosome is a 33-character binary string. Its layout is fixed:

| Segment | Bit count | Encoded value |
|---|---:|---|
| Bits 1–18 | 18 | `x1` |
| Bits 19–33 | 15 | `x2` |
| Total | 33 | The pair \((x_1,x_2)\) |

The number of possible bit strings is:

\[
2^{33}=8{,}589{,}934{,}592
\]

The first segment represents an integer from 0 to \(2^{18}-1=262143\). The second represents an integer from 0 to \(2^{15}-1=32767\).

The initial implementation creates a chromosome one random bit at a time:

```sql
SET @dna = '';
SET @i = 0;

WHILE @i < 33
BEGIN
    SET @dna = @dna
        + CAST(CAST(RAND() * 2 AS int) AS varchar(1));
    SET @i = @i + 1;
END;
```

It is intentionally easy to read, although it is not an efficient population-generation technique for large runs.

## From bits to real values

The original post used `dbo.ConvertFromBase`, a scalar UDF credited in that post to D. Patrick Caldwell, to convert binary fragments to decimal integers.

Conceptually, for a binary string of length \(L\), the conversion is:

\[
\text{integer}=\sum_{i=0}^{L-1}b_i2^i
\]

The original script then scales each decoded integer to the target interval:

```sql
SET @x1 = -3 + (@x1 * 0.00005760);
SET @x2 = 4.1 + (@x2 * 0.0000518);
```

The logic is correct, but the constants are manually rounded. This means the all-ones chromosome does not map exactly to the stated upper bounds. The intended formula is:

\[
x=x_{\min}+\frac{n}{2^L-1}(x_{\max}-x_{\min})
\]

where \(n\) is the decoded integer.

The revised script calculates that transformation from the bounds and bit widths:

```sql
SET @x1 = @X1Min
        + @X1Integer * (@X1Max - @X1Min) / @X1Levels;

SET @x2 = @X2Min
        + @X2Integer * (@X2Max - @X2Min) / @X2Levels;
```

This makes the relationship inspectable and guarantees that an all-zero chromosome maps to the lower bound and an all-one chromosome maps to the upper bound.

## Fitness evaluation

Each decoded candidate receives the objective-function value as fitness:

```sql
SET @Fitness = 21.5
    + @X1 * SIN(4.0 * PI() * @X1)
    + @X2 * SIN(20.0 * PI() * @X2);
```

The original script used a literal approximation, `3.14159`; the revised version uses `PI()`.

Fitness is deterministic: the same chromosome always produces the same `x1`, `x2`, and fitness. This makes the example much simpler than Bill Talada’s robot-and-cans implementation, where each candidate must be evaluated through a stateful simulation.

That simplification is both the strength and the limit of the example:

- It makes the GA mechanics accessible and easy to validate.
- It does not teach as much about simulation, policies, state transitions, or emergent behavior.

## Selection, crossover, and mutation

### Two-elite selection

The historical implementation selects only the two highest-fitness individuals. Every other member of the population is discarded, and all new offspring descend from these two parents.

This is easy to express in T-SQL and creates a compact script. It also creates strong selection pressure and a serious diversity bottleneck.

| Advantage | Cost |
|---|---|
| Simple ranking logic | Very small effective breeding population |
| Fast exploitation of strong candidates | High risk of premature convergence |
| Easy to explain in a blog post | Mutation becomes the main source of novelty |
| Few implementation choices | Local maxima can dominate early |

The original code used several `MAX()` subqueries. The revised version uses a single `ROW_NUMBER()` ranking with deterministic tie-breaking:

```sql
ROW_NUMBER() OVER (
    ORDER BY fitness DESC, sequence_id ASC
)
```

This ensures that the retained parents and the DNA used to create offspring are the same ranked individuals, even when fitness values tie.

### One-point crossover

The child chromosome is built from a prefix of parent 1 and a suffix of parent 2:

```sql
SET @ChildDNA = LEFT(@Parent1, @CutPosition)
    + RIGHT(@Parent2, @ChromosomeLength - @CutPosition);
```

The crossover point can fall inside the `x1` bits, inside the `x2` bits, or at their boundary. The child can therefore inherit part of the binary encoding of either variable from both parents.

### Mutation

The 2010 script performs five random bit-replacement attempts per child. A replacement is not necessarily a true mutation: the selected bit may already have the replacement value, and the same position can be selected more than once.

The revised script uses an explicit bit flip with configurable probability per locus:

```sql
IF RAND() < @MutationRate
BEGIN
    SET @ChildDNA = STUFF(
        @ChildDNA,
        @Position,
        1,
        CASE SUBSTRING(@ChildDNA, @Position, 1)
            WHEN '0' THEN '1'
            ELSE '0'
        END
    );
END;
```

With a 33-bit chromosome, a starting rate of \(1/33\) means roughly one expected bit flip per child. It is a transparent starting point, not a universal optimum.

## What the original teaches—and where it breaks down

The original post remains valuable because it exposes design choices that are often hidden in textbook pseudocode.

### What it teaches well

- A GA needs a representation, not just a fitness function.
- A chromosome can encode a real-valued solution indirectly.
- Fitness connects the encoded candidate to the problem objective.
- Selection, crossover, and mutation can be expressed with ordinary database-language constructs.
- A table can make genotype, phenotype, and fitness visible in one place.

### Where it is deliberately weak

- **Diversity:** only two parents produce every new candidate.
- **Mutation semantics:** random replacement is not the same as a guaranteed flip.
- **Numerical clarity:** rounded scaling coefficients hide the relationship between integer range and continuous bounds.
- **Reproducibility:** repeated calls to `RAND()` without experiment-level seed management make exact reruns difficult.
- **Performance:** loops, string manipulation, and scalar UDF calls are not a scalable numerical-computing pattern.
- **Observability:** a final best value alone does not reveal whether the population converged prematurely.

The purpose of the revised version is not to erase these limitations. It is to make them explicit and correct the narrowest issues while preserving the shape of the original exercise.

## Minimal 2026 revision

The revised script is intentionally modest. It does not turn the example into a modern evolutionary-optimization framework.

It improves the following points:

| Area | Historical approach | Minimal revision |
|---|---|---|
| Domain scaling | Rounded multipliers | Formula derived from bounds and bit widths |
| Pi | `3.14159` literal | `PI()` |
| Parent ranking | Repeated `MAX()` subqueries | Stable `ROW_NUMBER()` ranking |
| Tie handling | Implicit and potentially inconsistent | Explicit `fitness DESC, sequence_id ASC` order |
| Mutation | Five random replacements | Configurable per-bit flip probability |
| Elitism | Two parents survive conceptually | Two elite rows are copied into the next generation |
| Measurement | Final output only | Best, mean, standard deviation, and unique DNA per generation |
| Parameters | Scattered constants | Central configuration block |

The revised script still intentionally uses a two-elite lineage. That is not the selection model I would choose for a robust optimizer; it remains there so that the relationship with the historical code is clear.

A natural next experiment would compare this elite-only approach with tournament selection or rank-based selection under the same population size and evaluation budget. The useful metrics are not just best fitness but also diversity, convergence speed, variance across repeated runs, and sensitivity to mutation rate.

## Why this is not production architecture

T-SQL can express the mechanics of a GA. That does not mean a database engine is the best primary runtime for evolutionary computation.

For a real workload, the separation would normally look like this:

| Responsibility | Typical place |
|---|---|
| Source data, business constraints, and result storage | SQL Server |
| Relational calculations and data-heavy fitness components | SQL Server |
| Evolutionary loop, selection, crossover, and mutation | Python, .NET, or R |
| Numerical optimization | SciPy, DEAP, pymoo, Nevergrad, Optuna, or a suitable .NET library |
| Experiment tracking and audit trail | SQL Server plus version control and artifact storage |

This division uses SQL where it is strongest: durable data, relational operations, governed access, constraints, reporting, and auditability. It uses a general-purpose or scientific runtime where it is strongest: stochastic algorithms, numerical libraries, parallel computing, testing, and visualization.

The scalar conversion function is also a useful reminder of changing platform context. SQL Server can inline eligible scalar UDFs in modern compatibility levels, but that behavior is conditional and should not be assumed as a universal performance fix. The original procedural pattern remains educational rather than a throughput-oriented design.

## Conclusion

This experiment is worth preserving because it makes a genetic algorithm concrete for SQL developers. It turns abstract concepts—chromosome, phenotype, fitness, selection, crossover, mutation, generation—into familiar T-SQL objects and statements.

Its limitations are equally instructive. Selecting only two parents shrinks diversity. Random replacement is not the same as explicit mutation. Rounded scale factors obscure numerical intent. And a database procedural loop is not automatically an appropriate optimization runtime.

The most useful 2026 reading of the script is therefore neither nostalgia nor a claim that “SQL can do everything.” It is an example of technical judgment over time: preserve the original purpose, state its limits, improve what can be improved without rewriting history, and choose different architecture when the real problem demands it.

## Credits and provenance

I wrote the original code in 2010 as part of my final project for UNED’s postgraduate programme **[Aprendizaje Estadístico y Data Mining (Plan 09)](https://formacionpermanente.uned.es/tp_actividad/actividad/aprendizaje-estadistico-y-data-mining-plan-09)**.

I later published the script as **“T-SQL y Algoritmos Genéticos”** on **sqlast.blogspot.com**.

That post explicitly stated that its high-level evolutionary approach was based on Bill Talada’s **“A Genetic Algorithm Sample in T-SQL,”** published by SQLServerCentral in January 2010. Talada’s example evolves a robot policy for the robot-and-cans problem described by Melanie Mitchell in *Complexity: A Guided Tour* (2009). My original post adapted the broad mechanism to the two-variable continuous optimization problem discussed here.

The `ConvertFromBase` function in the original post was attributed to D. Patrick Caldwell’s base-conversion example. The historical script preserves that attribution. See the repository [NOTICE](./NOTICE.md) for the scope of the repository’s licensing and provenance statements.
