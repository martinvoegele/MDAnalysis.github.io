---
layout: post
title: "GSOC 2018: Improvements in distance search methods"
---

We are pleased to announce another successful year of [Google Summer of Code][] with the [NumFOCUS][] organization,
thanks to [Richard Gowers][] and [Jonathan Barnoud][] for mentoring the GSoC students.
This year one of the projects was [to improve the performance of pairwise distance computations][project], which is used quite frequently in MDAnalysis in different forms.

MDAnalysis v0.19.0 and higher now include the _new functions [`MDAnalysis.lib.distances.capped_distance`][] and [`MDAnalysis.lib.distances.self_capped_distance`][]_
which offer a much faster way to calculate all pairwise distances up to a certain maximum distance.
By only considering distances up to a certain maximum, we can use various algorithms to optimise the number of pairwise comparisons that are performed.
Behind the scenes, these functions are using one of three different algorithms:
[bruteforce][] which is a naive pairwise distance calculation algorithm,
[pkdtree][] which is a wrapper method around Scipy's KD tree search algorithm
and [nsgrid][] which is an implementation of cell-list algorithm.
This last algorithm uses the new [`MDAnalysis.lib.nsgrid`][] module which was implemented with the help of [Sebastien Buchoux][].

For more information on these algorithms the reader is encouraged to read @ayushsuhane's [blog], which includes a comparison of these approaches and their performance in different conditions.


## The GSoC project

One of the major bottleneck in various analysis routines in MDAnalysis (and typically in molecular dynamics studies) is the evaluation of pairwise distances among the particles.


The primary problem revolves around fixed radius neighbor search algorithms.
MDAnalysis offers a suite of algorithms including brute force method, tree-based binary search algorithms to solve such problems.
While these methods are suitable for a variety of analysis functions using pairwise distances in MDAnalysis, one of the question was whether one can improve the performance of distance calculations using other established neighbor search methods.

This question led to the inception of a Google Summer of Code [project][] with [NumFOCUS][].
[Ayush Suhane][] completed the project and was able to demonstrate performance improvements for specific cases of distance selections, identification of bonds, and radial distribution function in the analysis module of MDAnalysis.
More details on the commit history, PR's and blog posts can be found in the final [report][] submitted to GSoC. Real-time benchmarks for specific modules in MDAnalysis can be found [here](https://www.mdanalysis.org/benchmarks/). 


## The new `capped_distance()` function

The major highlight of the project is the introduction of [`MDAnalysis.lib.distances.capped_distance`][] which allows automatic selection of methods based on predefined set of rules to evaluate pairs of atoms in the neighborhood of any particle. It allows a user-friendly interface for the developers to quickly implement any new algorithm throughout MDAnalysis modules. To test any new algorithm, one must comply with the following API:

```python
def newmethod_capped(reference, configuration, max_cutoff, min_cutoff=None, box=None, return_distance=True):
    """
        An Algorithm to evaluate pairs between reference and configuration atoms
        and corresponding distances
    """
    return pairs, distances
```

Once the method is defined, register the function name in ``_determine_method`` in ``MDAnalysis.lib.distances`` as:

```python
from MDAnalysis.lib import distances
distances_methods['newmethod'] = newmethod_capped
```
That's it. The new method is ready to be tested across functions which use ``capped_distance``. For any specific application, it can be called as ``capped_distance(ref, conf, max_dist, method='newmethod')`` from the function.


## Treatment of periodic boundary conditions

While implementing any new algorithm for Molecular dynamics trajectories, one additional requirement is to handle the periodic boundary conditions.
A combination of versatile function ``augment_coordinates`` and ``undo_augment`` can be used with any algorithm to handle PBC. 
The main idea is to extend the box by generating duplicate particles in the vicinity of the box by ``augment_coordinates``. 
These duplicates, as well as the original particles, can now be used with any algorithm to evaluate the nearest neighbors. 
After the operation, the duplicate particles can be reverted back to their original particle indices using ``undo_augment``. 
These functions are available in ``MDAnalysis.lib._augment``. We encourage the interested readers to try different algorithms using these functions.
Hopefully, you can help us improve the performance further with your feedback. As a starting point, the skeleton to enable PBC would take the following form:

```python
def newmethod_search(coords, centers, radius, box=None):
    aug, mapping = augment_coordinates(coords, box, radius)
    all_coords = no.concatenate([coords, aug])
    """
        Perform operations for distance evaluations
        with **all_coords** using the new algorithm 
        and obtain the result in indices
    """
    indices = undo_augment(indices, mapping, len(coords))
    return indices
```

Finally, this function can be tested with ``capped_distance`` to check the performance against already implemented algorithms in MDAnalysis.

## Performance improvements

As a final note, we managed to get a speed improvement of 
- ~ 2-3 times in Radial Distribution Function computation, 
- ~ 10 times in identification of bonds, and 
- ~ 10 times in distance based selections for the already existing benchmarks in MDAnalysis. 

The performance is also found to improve with larger datasets but is not reported in benchmarks. Any motivated reader is welcome to submit their feedbacks about the performance of the above-mentioned functions on their data, and/or a benchmark which we would be happy to showcase to the world.

This was a flavor of what work was done during GSoC'18. Apart from performance improvements, it is envisioned that this internal functionality will reduce the burden from the user to understand all the technical details of distance search algorithms and instead allow a user to focus on their analysis, as well as allow future developers to easily implement any new algorithm which can exceed the present performance benchmarks.


— [Ayush Suhane][], [Richard Gowers][]

[Google Summer of Code]: https://summerofcode.withgoogle.com/projects/#5050592943144960 
[NumFOCUS]: https://numfocus.org/
[Ayush Suhane]: https://github.com/ayushsuhane
[project]: {{ site.baseurl }}{% post_url 2018-04-26-gsoc-students %}#ayush-suhane-improve-distance-search-methods-in-mdanalysis
[`MDAnalysis.lib.distances.capped_distance`]: https://www.mdanalysis.org/docs/documentation_pages/lib/distances.html#MDAnalysis.lib.distances.capped_distance
[`MDAnalysis.lib.distances.self_capped_distance`]: https://www.mdanalysis.org/docs/documentation_pages/lib/distances.html#MDAnalysis.lib.distances.self_capped_distance
[report]: https://gist.github.com/ayushsuhane/fd114cda20e93b0f61a8acb6d25d3276
[bruteforce]: http://www.csl.mtu.edu/cs4321/www/Lectures/Lecture%206%20-%20Brute%20Force%20Closest%20Pair%20and%20Convex%20and%20Exhausive%20Search.htm
[pkdtree]: https://en.wikipedia.org/wiki/K-d_tree
[nsgrid]: https://en.wikipedia.org/wiki/Cell_lists
[blog]: https://ayushsuhane.github.io/
[`MDAnalysis.lib.nsgrid`]: https://www.mdanalysis.org/docs/documentation_pages/lib/nsgrid.html
[Sebastien Buchoux]: https://github.com/seb-buch
[Richard Gowers]: https://github.com/richardjgowers
[Jonathan Barnoud]: https://github.com/jbarnoud
