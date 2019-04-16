[Advanced Assignments](./) > [DataBass](./databass)

## AA4: Join Ordering Optimization


* Released: Apr 16	
* Due: May 2 10AM
* Value: 5% of your grade
* Max team of 2

This assignment is part of the [DataBass Series of Advanced Assignment](./).  Take a look at the [README](./) in this directory to get an overview of the system and how to set it up.  Then read these instructions.  Since these advanced assignments are new and experimental, please be patient with us and [ask any questions on Piazza](https://piazza.com/class/jgwnwiy186d6pu).

In this assignment, you will use simple statistics to implement a simple version of the Selinger join optimizer to determine the join ordering for multi-table joins.
To do so, take a look at `optimizer.py`.  The class performs a single optimization, which is to replace the N-way `From` operator with a join tree that can actually be executed.
If you run the file, it executes a super naive implementation that generates a left-deep join plan consisting of cross-products (Theta Joins with join predicate of `True`).   

The file also contains a class called `SelingerOpt` that encapsulates the optimization that you will implement.  Specifically, you will implement the cost and cardinality estimations, as well as the bottom-up search algorithm as described in class.  We have marked the locations that you will need to edit `XXX:` comments.

If you are interested, cockroach labs recently wrote a [blog post about the SQL optimizer they wrote from scratch](https://www.cockroachlabs.com/blog/building-cost-based-sql-optimizer/).


#### Selinger

Recall that selinger, as described in class, is an iterative algorithm that first tries all 2-way joins and picks the best.  It then tries all the ways to add a third table and pikcs the best.  Then a fourth, and so on.  

For simplicity, you can assume that each tuple is one unit of cost, when computing the cost of a join. Thus the cost of `A join B` where A has 5 tuples and B has 10, is `5 + 5*10`.   To penalize cross products and large cardinality outputs, you will see that we add the output cardinality to the cost estimate.

In this assignment we will make a number of simplifying assumptions:

* All sources have explicit aliases and the only sources are table scans  (e.g., not subqueries).  

      # data does NOT have explicit alias, can assume this will not happen
      SELECT * FROM data    
      
      # Source is a subquery can assume this will not happen
      SELECT * FROM (SELECT 1) AS T
      
      # data has an explicit alias of T
      SELECT * FROM data AS T

* All attributes also reference the table.  

      # attribute a does not reference its table T, can assume this will not happen
      SELECT * FROM data as T WHERE a = 1 
      
      # can assume it will always referenc the table 
      SELECT * FROM data as T WHERE T.a = 1 

* We only consider equality join predicates between one attribute from each table:

      # This is ok
      A.a = B.b  
      
      # This will not happen because the right side is an expression
      A.a = B.b + 1

* The only access method is a table scan (no indexes)
* The only join algorithm is tuple-nested loop join
* Each attribute's values are distributed uniformly. 
  * If the attribute is numeric, then its values are distributed uniformly between the minimum and maximum values.  
  * If the attribute is non-numeric (string), it is uniformly distributed across the distinct values.

## The Assignment


#### Part 1: Statistics

To implement our simple version of Selinger, you will need to compute two statistics by editing the `Stat` class definition in `db.py`.   

1. The first is to compute the cardinality of the table, and store it in `Stat.card` in the constructor.  Since DataBass loads the full table into memory, you can compute an exact estimate.     
2. The second is to compute the domain of a given field in `Stat.__getitem__()`.    The method should return a dictionary: `dict(min=?, max=?, ndistinct=?)`
  * For a numeric field (see `Table.type()`), it should return min and max value in the field, along with the number of distinct values.
  * For a string field, it should set min and max to None, and set the number of distinct values.


#### Part 2: Enable the Selinger Optimization Code

Now uncomment the code in `Optimizer.expand_from_op()` so that the optimizer uses the Selinger optimization class.   At this point, you can run `python optimizer.py` to see its output.

We have implemented the core components of the Selinger optimization.  The algorithm takes as input the list of join predicates and a list of Sources (base Scan operators) that need to be joined.  It will return a left-deep join plan consisting of ThetaJoin operators.

Each predicate is a binary `Expr` object where the `l` and `r` attributes are `Attr` objects that specify the Source alias and attribute name that participates in the join.  For instance, a predicate may be `A.a = B.b`, where `Attr("a", "A")` represents the left side of the expression and `Attr("b", "B")`.  

Each source in passed to `SelingerOpt.__call__()` is a Scan operator that specifies the base table name (as can be found in the `Database`), as well as an alias for the table.    The latter is what attributes in join predicates refer to.

To help you with the assignment we have implemented a topdown join ordering algorithm `SelingerOpt.best_plan_topdown()` that recursively computes a good join plan using a depth-first approach.   It considers many unnecessary join orderings and revisits some join subplans many times, thus is less efficient than the implementation you will write. 

#### Part 3: Estimate Join Cardinalities

The top down implementation relies on cost and cardinality estimates to decide which plans are good, however we have not implemented the `SelingerOpt.cost()` and `SelingerOpt.card()` methods.  Edit these methods to estimate the appropriate statistics.  

#### Part 4: Selinger

Now that you can estimate join cardinalities, you can implement the selinger join ordering algorithm in `SelingerOpt.best_plan()`    

To do so, first implement `SelingerOpt.best_initial_join()`, which identifies the 2-table join with the lowest cost.  This initializes the left deep join plan, and you will then edit `best_plan()` to iteratively add additional joins to the plan.

### Part 5: Checking

Go to the root of your databass codebase repository. Run `PYTHONPATH=. python test/aa4.py`. The group truth implementation of all the three test in aa4.py cost about 1.4s. If your implementation is much slower than the ground truth implementaion, you will lose partial of the credits. 


### Submission Instructions


Go to the root of your databass codebase repository, and run `git pull` to make sure you have the most recent version.  You should see a submission script `submit.py`.

Run the submission script `submit.py` to package and zip your file.  It will ask you to submit you and your partners' UNIs and specify the assignment. It will create a zip file in the current directory.  

        python submit.py --help

After compression, submit to [dropbox](https://www.dropbox.com/request/GDl6CuTNeeNfRRMJBshG) . Both of the partners should submit the assignment separately. This time, no points will be awarded for code that does not compile. No points will be awarded to you if only your partner submits.
