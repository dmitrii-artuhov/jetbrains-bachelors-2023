# Test suite minimization for SE/CT

## Task 1

> What do you think are pros and cons of the two proposed ways of addressing the problem?

1. **Test suit minimization module**:
   - Pros:
     - We can choose from variety of different minimization algorithms as well as different coverage criteria in their internals.
     - Overall separate module might be more flexible solution for the end users. 
   - Cons:
     - Minimization algorithms might lead to a significant decrease in fault detection capabilities of the minimized test suite.
     - Some work of Kex itself is done for nothing (there are tests that are definitely not going to be in the minimized test suite, but still generated).
2. **Kex analysis process improvement**:
    - Pros:
      - Tests generation becomes faster.
      - We can cache the parts of the code for which tests have already been generated: subsequent Kex runs might only generate tests for newly created code.
      - Before generating particular test we can add meta-information that would clarify its purpose.
    - Cons:
      - We do not have an ability to choose from many different algorithms of eliminating tests (since we do not generate all the tests), so we limited to choosing code coverage criteria (but other heuristics might be possible).
      - Generated tests amount might still be big even after improvement.

## Task 2

> Which of the two methods, in your opinion, will perform better in terms of efficiency and quality? What could the efficiency and quality of them possibly depend on?

In my opinion, Kex analysis improvement is an easier feature to implement in general, but it is an open question for heuristics that might bring additional improvements. On the other hand, test suit minimization module brings more space for research and comparison (see pros), but is a more complex feature.

In terms of ending result (smaller tests redundancy, less compiling and running times) test suit minimization module seems to be more promising, because of its flexibility.

## Task 3

> You are given a set of items S. Each item is identified by its unique name and has some positive integer price. Write a function that will divide an original set into two subsets A and B such that:
> - abs(size(A) - size(B)) == size(S) % 2
> - the difference between the sums of prices of elements of subsets A and B is minimal

A dumb solution would be to write a recursion with cuts that would iterate over all possibilities for the elements in set A and calculate the minimum sum difference. Time complexity `O*(2^(n / 2))`. 