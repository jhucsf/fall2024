---
layout: mathjax
title: "Assignment 4: Parallel Quicksort"
---

**Due**: Friday, Nov 8th by 11:59 pm

Assignment type: **Pair**, you may work with one partner

# Parallel Quicksort

In this assignment, you will

1. Complete a partially-implemented program which uses the
   [Quicksort](https://en.wikipedia.org/wiki/Quicksort) sorting algorithm
   to sort a file containing a sequence of 64-bit signed integers;
   specifically, add code to use the [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)
   system call to create a shared mapping of the data in the file so that
   the contents of the file appear as an array in memory
2. Use child processes to execute the recursive calls in parallel, in order
   to take advantage of multiple CPU cores to complete sorting more
   quickly

## Grading Criteria

Your assignment grade will be determined as follows:

* Handling command line arguments, using open and mmap to map the file: 20%
* Parallel sorting using subprocesses: 50%
* Experiments and report: 15%
* Error reporting: 5%
* Design and coding style: 10%

## Getting Started

Download [csf\_assign04.zip](csf_assign04.zip) and unzip it. You will be
modifying the file `parsort.c`.

# Quicksort

The starter code has a correct implementation of quicksort, so you don't really
need to completely understand it in order to do the assignment. However, quicksort
is a very simple and elegant algorithm, so it's not too hard to explain.

The basic idea of quicksort is based on *partitioning* a sequence into
"left" and "right" partitions. Partitioning works as follows:

1. An arbitrary "pivot" element in the sequence is chosen
2. The sequence is re-arranged so that all of the elements less than
   the pivot value occur before the pivot value (this is the
   "left" partition), and all the elements
   greater than or equal to the pivot value occur after the pivot
   value (this is the "right" partition)

Once a sequence is partitioned, it can be sorted by recursively sorting
the left partition and the right partition.

Let's say that we want to sort all elements of a sequence called *arr*
between the indices *start* (inclusive lower bound) and *end*
(exclusive upper bound.) A pseudo-code implementation of quicksort
might look something like this:

```
quicksort( arr, start, end ) {
  n = end - start
  if ( n < 2 )
    // base case: sequence is trivially already sorted
    return

  // the partition function returns the index where
  // the pivot element ended up
  mid = partition( arr, start, end )

  quicksort( arr, start, mid )
  quicksort( arr, mid + 1, end )
}
```

Paritioning a sequence with $$n$$ elements is $$O(n)$$. Assuming that
quicksort generally chooses a pivot element such that the left and right
partitions are roughly equal in size, the overall running time of
quicksort is $$O(n \log n)$$ in the average case.

## Parallel quicksort

Because the two recursive calls to quicksort operate on completely independent
parts of the array, there is no reason why they can't execute in parallel
on different CPU cores.

Eventually we will learn about [threads](https://en.wikipedia.org/wiki/Thread_(computing))
as a mechanism for executing instructions on multiple CPU cores. For this assignment,
we will use [child processes](https://en.wikipedia.org/wiki/Child_process) to
execute the recursive calls in parallel. Normally, a data structure in a parent
process, such as an array being sorted, would not be shared with a child process,
since by default, child processes do not share memory with their parent process.
However, if a shared file mapping is created, the memory containing the data
of the mapped file is shared between parent and child processes, which means
a child process can participate in sorting the data in the file.
