# DuckDB Tutorial
This tutorial is composed of two exercises. In the first exercise, students will compare DuckDB,SQLite and Pandas in terms of performance and usability. The second exercise is about playing around with query execution and the query optimizer in DuckDB.

### Requirements
For these exercise, you will need a [Google Colab Account](https://colab.research.google.com/).

## Part (1) Using DuckDB

### Task
Download the .ipynb file from [here](https://raw.githubusercontent.com/pdet/duckdb-tutorial/master/Part%201/Exercise/Exercise.ipynb) and upload it as a Python 3 Notebook into [Google Colab](https://colab.research.google.com/).
Follow the steps depicted in the python notebook, and compare the performance of these 3 engines on three different tasks. You will load the data, execute different queries (focusing in selections, aggregations and joins) and finally will perform transactions cleaning dirty tuples from our dataset.

### Project Assignment
Similar to the task described above, you must download the .ipynb file from [here](https://raw.githubusercontent.com/pdet/duckdb-tutorial/master/Project/NYC_Cab_DuckDB_Assignment.ipynb) and upload it as a Python 3 Notebook into [Google Colab](https://colab.research.google.com/). In this assignment, you will experiment with the NYC Cab dataset from 2016. This dataset provides information (e.g., pickup/dropoff time, # of passengers, trip distance, fare) about cab trips done in New York City during 2016. You can learn more about the dataset clicking [here!](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

You will load this dataset into pandas, sqlite, and duckdb. You will compare the performance of multiple data-science like queries, including performing a fare estimation (i.e., predicting how much a ride will cost depending on distance) using machine learning.

In the first section you will implement the loader in duckdb **[5 points].**

The second section has two data-science like queries, the implementation in pandas is already given, and you should use it as a logical/correctness reference to write the queries for sqlite and duckdb, remember to compare the performance of the three different systems **[25 points]**.

Finally, in the third section you will implement a simple machine learning algorithm in duckdb to predict fare costs. A full implementation of pandas is given and a partial of sqlite. Again, use them as a logical/correctness reference and compare the performance of the three different systems. **[40 points]**

Remember to submit your notebook with the answers to all sections as well as a PDF document (max two papes) listing all experienced execution times and reasoning about the performance difference in these systems.


## Part (2) Query Optimization

### Task
Download the .ipynb file from [here](https://raw.githubusercontent.com/pdet/duckdb-tutorial/master/Part%202/DuckDB_Exercise2.ipynb) and upload it as a Python 3 Notebook into [Google Colab](https://colab.research.google.com/). Follow the instructions in the notebook, and try to find a way to formulate the SQL query so that the query execution matches, or even exceeds the performance in DuckDB by serving as the manual query optimizer!

### Local Execution
For those who would rather run the tutorial locally, below are the contents of the IPython notebook repeated:

#### Setup
First we need to install DuckDB:
```bash
pip install duckdb --pre
```

#### Loading The Data
```python
import urllib.request
import zipfile

print("Downloading datasets")

urllib.request.urlretrieve("https://github.com/Mytherin/datasets/raw/main/tpch_sf01.zip", "tpch_sf01.zip")

print("Decompressing files")

with zipfile.ZipFile("tpch_sf01.zip","r") as zip_ref:
	zip_ref.extractall("./")

print("Finished.")
```

#### Load Data in DuckDB
```python
import duckdb
con = duckdb.connect(':memory:')

queries = []
with open('tpch_sf01/schema.sql', 'r') as f:
  queries += [x for x in f.read().split(';') if len(x.strip()) > 0]
with open('tpch_sf01/load.sql', 'r') as f:
  queries += [x for x in f.read().split(';') if len(x.strip()) > 0]

print("Beginning data load")
for q in queries:
  con.execute(q)
print("Finishing data load")
```

#### Inspecting the Query Plan
The query plan of a query can be inspected by prefixing the query with`explain`. By default, only the physical query plan is returned. You can use `PRAGMA explain_output='all'` to output the unoptimized logical plan, the optimized logical plan and the physical plan instead

```python
def explain_query(query):
  print(con.execute("EXPLAIN " + query).fetchall()[0][1])

query = """
SELECT l_orderkey, SUM(l_extendedprice)
FROM lineitem
WHERE l_discount < 5
GROUP BY l_orderkey
ORDER BY l_orderkey DESC;
"""

explain_query(query)
```

#### Profiling Queries
Rather than only viewing the query plan, we can also run the query and look at the profile output. The function `run_and_profile_query` below performs this profiling by enabling the profiling, writing the profiling output to a file, and then printing the contents of that file to the console.

The profiler output shows extra information for every operator; namely how much time was spent executing that operator, and how many tuples have moved from that operator to the operator above it.

For a `SEQ_SCAN` (sequential scan), for example, it shows how many tuples have been read from the base table. For a `FILTER`, it shows how many tuples have passed the filter predicate. For a `HASH_GROUP_BY`, it shows how many groups were created and aggregated.

These intermediate cardinalities are important because they do a good job of explaining why an operator takes a certain amount of time, and in many cases these intermediates can be avoided or drastically reduced by modifying the way in which a query is executed.


```python
def run_and_profile_query(query):
  con.execute("PRAGMA enable_profiling")
  con.execute("PRAGMA profiling_output='out.log'")
  con.execute(query)
  with open('out.log', 'r') as f:
    output = f.read()
  con.execute("PRAGMA disable_profiling")
  print(output)

query = """
SELECT l_orderkey, SUM(l_extendedprice)
FROM lineitem
WHERE l_discount < 5
GROUP BY l_orderkey
ORDER BY l_orderkey DESC;
"""

run_and_profile_query(query)
```

#### Query Optimizations
An important component of a database system is the optimizer. The optimizer changes the query plan so that it is logically equivalent to the original plan, but (hopefully) executes much faster.
In an ideal world, the optimizer allows the user not to worry about how to formulate a query: the user only needs to describe what result they want to see, and the database figures out the most efficient way of retrieving that result.
In practice, this is certainly not always true, and in some situations it is necessary to rephrase a query. Nevertheless, optimizers generally do a very good job at optimizing queries, and save users a lot of time in manually reformulating queries.
Let us run the following query and see how it performs:

```python
query = """
SELECT
    l_orderkey,
    sum(l_extendedprice * (1 - l_discount)) AS revenue,
    o_orderdate,
    o_shippriority
FROM
    customer,
    orders,
    lineitem
WHERE
    c_mktsegment = 'BUILDING'
    AND c_custkey = o_custkey
    AND l_orderkey = o_orderkey
    AND o_orderdate < CAST('1995-03-15' AS date)
    AND l_shipdate > CAST('1995-03-15' AS date)
GROUP BY
    l_orderkey,
    o_orderdate,
    o_shippriority
ORDER BY
    revenue DESC,
    o_orderdate
LIMIT 10;
"""

run_and_profile_query(query)
```

#### Manual Query Optimizations


In order to get a better idea of how query optimizers work, we are going to perform *manual* query optimization. In order to do that, we will disable all query optimizers in DuckDB, which means the query will run *as-is*. We can then change the way the query is physically executed by altering the query. Let's try to disable the optimizer and looking at the query plan:

```python
con.execute("PRAGMA disable_optimizer")
explain_query(query)
```

Looking at the plan you now see that the hash joins that were used before are replaced by cross products followed by a filter. This is what was literally written in the query, however, cross products are extremely expensive! We could run this query, but because of the cross products it will take extremely long.

Let's rewrite the query to explicitly use joins instead, and then we can actually run it:

```python
query = """
SELECT
    l_orderkey,
    sum(l_extendedprice * (1 - l_discount)) AS revenue,
    o_orderdate,
    o_shippriority
FROM
    customer
    JOIN orders ON (c_custkey=o_custkey)
    JOIN lineitem ON (l_orderkey=o_orderkey)
WHERE
    c_mktsegment = 'BUILDING'
    AND o_orderdate < CAST('1995-03-15' AS date)
    AND l_shipdate > CAST('1995-03-15' AS date)
GROUP BY
    l_orderkey,
    o_orderdate,
    o_shippriority
ORDER BY
    revenue DESC,
    o_orderdate
LIMIT 10;
"""

run_and_profile_query(query)

```

##### Assignment

Now the query actually finishes; however, it is still much slower than before. There are more changes that can be made to the query to make it run faster. Your assignment (and challenge!) is to adjust the query so that it runs in similar speed to the query with optimizations enabled. You will be the human query optimizer replacing the disabled one.

Hint:

1. Join order matters!
2. DuckDB always builds the hash table on the *right side* of a hash join.
3. Filters? Projections?

Another important consideration is that the query optimization should still output the same query result! For that reason, you can use the function below to verify that your result is still correct after optimization.

```python
import datetime
expected_result_1 = [(223140, 355369.0698, datetime.date(1995, 3, 14), 0),
                    (584291, 354494.7318, datetime.date(1995, 2, 21), 0),
                    (405063, 353125.4577, datetime.date(1995, 3, 3), 0),
                    (573861, 351238.277, datetime.date(1995, 3, 9), 0),
                    (554757, 349181.7426, datetime.date(1995, 3, 14), 0),
                    (506021, 321075.581, datetime.date(1995, 3, 10), 0),
                    (121604, 318576.4154, datetime.date(1995, 3, 7), 0),
                    (108514, 314967.0754, datetime.date(1995, 2, 20), 0),
                    (462502, 312604.542, datetime.date(1995, 3, 8), 0),
                    (178727, 309728.9306, datetime.date(1995, 2, 25), 0)]
def profile_and_verify_query(query, expected_results):
  con.execute("PRAGMA enable_profiling")
  con.execute("PRAGMA profiling_output='out.log'")
  results = con.execute(query).fetchall()
  with open('out.log', 'r') as f:
    output = f.read()
  con.execute("PRAGMA disable_profiling")
  print(output)
  if len(expected_results) != len(results):
    print("Incorrect result, expected", expected_results, "but got", results)
    return False
  for r in range(len(results)):
    if len(results[r]) != len(expected_results[r]):
      print("Incorrect result, expected", expected_results, "but got", results)
      return False
    for c in range(len(results[r])):
      if results[r][c] != expected_results[r][c]:
        print("Incorrect result, expected", expected_results, "but got", results)
        return False
  return True

def profile_and_verify_1(query):
  return profile_and_verify_query(query, expected_result_1)
```

```python
query = """
SELECT
    l_orderkey,
    sum(l_extendedprice * (1 - l_discount)) AS revenue,
    o_orderdate,
    o_shippriority
FROM
    customer
    JOIN orders ON (c_custkey=o_custkey)
    JOIN lineitem ON (l_orderkey=o_orderkey)
WHERE
    c_mktsegment = 'BUILDING'
    AND o_orderdate < CAST('1995-03-15' AS date)
    AND l_shipdate > CAST('1995-03-15' AS date)
GROUP BY
    l_orderkey,
    o_orderdate,
    o_shippriority
ORDER BY
    revenue DESC,
    o_orderdate
LIMIT 10;
"""

run_and_profile_query(query)
```

##### Bonus Assignment
As a bonus assignment, here is another query that you can optimize. Note that this query is currently NOT fully optimized by DuckDB because of a problem in the query optimizer, hence on this query it is actually possible to not only match, but beat the DuckDB query optimizer!

```python
query = """
SELECT
    nation,
    o_year,
    sum(amount) AS sum_profit
FROM (
    SELECT
        n_name AS nation,
        extract(year FROM o_orderdate) AS o_year,
        l_extendedprice * (1 - l_discount) - ps_supplycost * l_quantity AS amount
    FROM
        part,
        supplier,
        lineitem,
        partsupp,
        orders,
        nation
    WHERE
        s_suppkey = l_suppkey
        AND ps_suppkey = l_suppkey
        AND ps_partkey = l_partkey
        AND p_partkey = l_partkey
        AND o_orderkey = l_orderkey
        AND s_nationkey = n_nationkey
        AND p_name LIKE '%green%') AS profit
GROUP BY
    nation,
    o_year
ORDER BY
    nation,
    o_year DESC;
"""


# HINT: first replace the cross products with joins before running this query!
# profile_and_verify_1(query)
```

## Extra: Writing a String scalar UDF
Now, let us help you writing your own SQL function in DuckDB. 

Let us create the function `INSTR(VARCHAR,VARCHAR) -> INTEGER             [ SQLite ]` that returns the index of the first occurence of a string needle in a string haystack. For example, consider the haystack string ‘Holland’. We want to know the first occurance of the next three needles ‘Ho’, ‘nd’ and ‘x’. They are expected to return 1, 6 and 0, respectively.

```bash
sqlite> create table strings(s VARCHAR);
sqlite> insert into strings values('Holland');
sqlite> select instr(s,'Ho') from strings;
1
sqlite> select instr(s,'nd') from strings;
6
sqlite> select instr(s,'x') from strings;
0
```

A list of functions from other systems that DuckDB is currently lacking can be found [here](https://github.com/cwida/duckdb/issues/193). If you implement the functions successfully, feel free to submit a pull request! 

### Requirements
DuckDB requires [CMake](https://cmake.org) to be installed and a `C++11` compliant compiler. GCC 4.9 and newer, Clang 3.9 and newer and VisualStudio 2017 are tested on each revision, but any `C++11` compliant compiler should work.

### Building
The source code can be downloaded from [the DuckDB repository](https://github.com/cwida/duckdb/commits/master). Run the following command to clone the github directory:
```bash
$ git clone https://github.com/cwida/duckdb.git
$ cd duckdb/
```

Alternatively, a zip file with the source code can be downloaded from [here](https://github.com/cwida/duckdb/archive/master.zip).

From now on in this tutorial, we assume you are in the `duckdb/` folder. 
This is optional, but you may also checkout to the same code branch that we had before coding the funciton of this tutorial, below:
```bash
git checkout a55686784d46fd85ff39a9b3a2b04745759c4dd1
```

After downloading the source code, the build must be initialized with CMake and the source files must be built. On Linux/OSX, the build can be done by simply using the command `make debug`. On Windows, you must use CMake to generate a Visual Studio project and then use Visual Studio to build that project. Building might take some time depending on how fast your computer is!

If you build using the `make debug` command, the build will be placed in the `build/debug` directory. A shell can be found in the location `build/debug/tools/shell/shell` from which you can issue simple commands to DuckDB. This shell is based on the `sqlite3` shell.

Now, you have your local copy of DuckDB and needs to create a code branch to do code your function. Suppose we are creating a branch named `instrfunction` and checkout to it:
```bash
duckdb$ git push --set-upstream origin instrfunction 
duckdb$ git checkout -b instrfunction
```

### Code your test
We start coding our function by its unit tests. This coding technique is called Test Driven Development (TDD) and is quite useful to shorten the development cycle.
We code the tests first and only later on we code the function trying to pass the tests.

In the test case, we code what is expected from the function in a succinct way.  For this tutorial, consider the code snippet below as our test case and later on we will try to write a function to pass its three assertions. 


```cpp
TEST_CASE("Instr test", "[function]") {
	unique_ptr<QueryResult> result;
	DuckDB db(nullptr);
	Connection con(db);
	con.EnableQueryVerification();

	REQUIRE_NO_FAIL(con.Query("CREATE TABLE strings(s VARCHAR, off INTEGER, length INTEGER);"));
	REQUIRE_NO_FAIL(con.Query("INSERT INTO strings VALUES ('hello', 1, 2)"));

	// Test first letter
	result = con.Query("SELECT instr(s,'h') FROM strings");
	REQUIRE(CHECK_COLUMN(result, 0, {1}));
}

```


In a very simplistic way, the test case is wrapped with the `TEST_CASE("Instr test", "[function]")` function. Its name `"Instr test"` is importat as we may execute this test case directly, as we will discuss shortly.

In the test case, we create a table and insert four rows. We may observe three assertions in the code snippet. Two of them require `NO FAIL` to setup our input test data. The third assertion is the simple return that we expect from the `INSTR()` function. For instance, we expect the return 1 for `INSTR('hello','h')` as 'h' is in the first string postion. Note that `CHECK_COLUMN` requires the result of the SQL command and the column to be asserted. In our test case, we test column `s` (or column 0).

For this tutorial the code snippet above is sufficient, but you may find our complete test cases for the `INSTR()` function [here](https://github.com/cwida/duckdb/blob/master/test/sql/function/test_instr.cpp).

### Execute your test case
We use the `lldb` system to run test cases and benchmarks. 
First, we run `make debug` to compile DuckDB in debug mode for testing. The debug executable tests can be found at `../build/debug/test/unittest`.

The code snippet shows the execution of our test case without breakpoints. To run it in `lldb`, we type `r "Instr test"`.

```bash
duckdb$ lldb ./build/debug/test/unittest 
(lldb) target create "./build/debug/test/unittest"
Current executable set to './build/debug/test/unittest' (x86_64).
(lldb) r "Instr test"
Process 4991 launched: 'build/debug/test/unittest' (x86_64)
[1/1] (100%): Instr test                                                        
===============================================================================
All tests passed (3 assertions in 1 test case)

Process 4991 exited with status = 0 (0x00000000) 
```


### Code your function signature

We are now in position to start coding the function. Here, you may see the function signature:
```cpp
static int64_t instr(string_t haystack, string_t needle);
```

Note that this function needs two inputs: the haystack and the needle.

DuckDB provides a great template to ease the inclusion of functions into its query processing engine. Our generic function “InstrOperator” calls two arguments required by the SQL function needle and haystack (or `left` and `right` arguments). The code looks like this:

```cpp
struct InstrOperator {
   template <class TA, class TB, class TR> static inline TR Operation(TA left, TB right) {
       return instr(left, right);
   }
};
```


### Implement the register function

Now that your function is ready, we need to inform DuckDB about it.
The query processing engine requires registering the new function, its arguments and return type. The code looks like this:

```cpp
void InstrFun::RegisterFunction(BuiltinFunctions &set) {
   set.AddFunction(ScalarFunction("instr", // name of the function
                                   {SQLType::VARCHAR,
                                   SQLType::VARCHAR}, // argument list
                                   SQLType::BIGINT, // return type int64_t
                                   ScalarFunction::BinaryFunction<string_t, string_t, int32_t, InstrOperator, true>));
}

```

You may see that we call the register function as “InstrFun” with four parameters: the name of the function (“instr”), the input arguments to match our function signature (the needle and the haystack), the return type also to match our function signature (the needle starting position) and the reference to the template.

Note that the template requires the two arguments (left, right) to match our SQL function. In this case, we call the  `ScalarFunction::BinaryFunction<...>` with the types of the two inputs.

### Including in the manifesto

Now that your function and register are ready, you need to include in the code manifesto. In the case of the SQL function, you need to include the reference to our function in three places.

####  In the manifesto of the folder where you coded your function you include the name of the source code. Our example is “instr.cpp”. In this example:  .../src/function/scalar/string/CMakeLists.txt

```cpp
add_library_unity(duckdb_func_string
                 OBJECT
                 reverse.cpp
                 ...
                 substring.cpp
                 instr.cpp)
```

#### In the manifesto of the string functions: .../src/function/scalar/string_functions.cpp

Like this: 

```cpp
void BuiltinFunctions::RegisterStringFunctions() {
  ...
   Register<SubstringFun>();
   Register<InstrFun>();
}
```

in the header at: .../src/include/duckdb/function/scalar/string_functions.hpp

Like this:

```cpp
namespace duckdb {
 
...
struct InstrFun {
   static void RegisterFunction(BuiltinFunctions &set);
};
```

### Code your mock

A good idea to make sure that everything is working properly, before implementing the function itself, is to mock the behavior of your function.
Remember that the test case expects a return `REQUIRE(CHECK_COLUMN(result, 0, {1}));` for the SQL code. In our case, we are just expecting `1`.

```cpp
static int64_t instr(string_t haystack, string_t needle) {
	return 1;
}
```

If you compile DuckDB in debug mode `make debug` and then test it with your mock, this is the expected result from `lldb`:

```bash
duckdb$ make debug
duckdb$ lldb ./build/debug/test/unittest 
(lldb) target create "./build/debug/test/unittest"
Current executable set to './build/debug/test/unittest' (x86_64).
(lldb) r "Instr test"
Process 4991 launched: 'build/debug/test/unittest' (x86_64)
[1/1] (100%): Instr test                                                        
===============================================================================
All tests passed (3 assertions in 1 test case)

Process 4991 exited with status = 0 (0x00000000) 
```


### Code your function
 Now that everything is working like a charm, it is time for you to code the function itself replacing the mock. 
 The complete source code may be found [here](https://github.com/cwida/duckdb/blob/master/src/function/scalar/string/instr.cpp).

```cpp
static int64_t instr(string_t haystack, string_t needle) {
	int64_t string_position = 0;

	// tons of coding

	return string_position;
}
```

Do not forget to test the function until it passes all the assertions!!!


### Benchmarking
This last step is optional, but very welcomed. A benchmark looks like a test case. In our case, we will benchmark the response time of our `INSTR()` function. 

The benchmark implementation is wrapped with the `DUCKDB_BENCHMARK()` function. Similar to the test case, the first argument is the benchmark name to be executed later on, say `StringInstr`. In the code below, we also generate a set of strings to be inserted in the database with a defined lenght of 4 chars.
The benchmark source code can be found [here](https://github.com/cwida/duckdb/blob/master/benchmark/micro/string.cpp).

```cpp
DUCKDB_BENCHMARK(StringInstr, "[string]")
STRING_DATA_GEN_BODY(4)
string GetQuery() override {
	return "SELECT INSTR(s1, 'h') FROM strings";
}
string BenchmarkInfo() override {
	return "STRING INSTR";
}
FINISH_BENCHMARK(StringInstr)
```

We execute the benchmark with `lldb` in a similar way to test cases.  We can run our specific benchmark by running the `benchmark_runner` program: `build/debug/benchmark/benchmark_runner  "StringInstr"`. The result may look like this:

```bash
duckdb$ lldb ./build/debug/benchmark/benchmark_runner 
(lldb) target create "./build/debug/benchmark/benchmark_runner"
Current executable set to './build/debug/benchmark/benchmark_runner' (x86_64).
(lldb) r StringInstr
Process 5095 launched: 'build/debug/benchmark/benchmark_runner' (x86_64)
-----------------
|| StringInstr ||
-----------------
Cold run...DONE
1/5...1.830888
2/5...1.841105
3/5...1.897842
4/5...2.038949
5/5...1.897553
Process 5095 exited with status = 0 (0x00000000) 
```

### Code formatting and Pull request
DuckDB cares about the source code formatting and provides some guidelines (identation, variables, etc) to help you out [here]().
DuckDB provides a python script to help formatting the source code according to the contributing guidelines.

```bash
duckdb$ python3 scripts/format.py master
``` 

If everything goes ok (pass tests, formatting and etc), you can create a pull request (PR) at the DuckDB github site. 
When you create a PR, Github will also run some code formattig checks for you. Before asking for a PR, do not forget to commit your changes to Git.


## (3) Writing an Integer scalar UDF
Here you may see another UDF using the same coding methodology: write test, then implement the UDF.

### Creating a Simple Function
For the example, we will create the following simple function: ```add_one(INTEGER) -> INTEGER```. This function adds one to its input value and returns the result. 

##### Creating the Tests
It is easiest to start developing by first creating tests, this allows us to already model the correct behavior and to later verify that our function achieves the correct behavior. The tests for functions are located in the `test/sql/function` directory. Navigate there, and create the file `test_add_one.cpp` to test our function. Then open the `CMakeLists.txt` in that same directory (`test/sql/function/CMakeLists.txt`) and add the file `test_add_one.cpp` to the to-be-built files.

In the test file, we can add the following snippet of code to test our function:
```cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;
using namespace std;

TEST_CASE("Test add one function", "[function]") {
	unique_ptr<QueryResult> result;
	DuckDB db(nullptr);
	Connection con(db);
	con.EnableQueryVerification();

	REQUIRE_NO_FAIL(con.Query("CREATE TABLE integers(i INTEGER)"));
	REQUIRE_NO_FAIL(con.Query("INSERT INTO integers VALUES (1), (2), (3), (NULL)"));

	// 1 + 1 = 2
	result = con.Query("SELECT add_one(1)");
	REQUIRE(CHECK_COLUMN(result, 0, {2}));
	// NULL + 1 = NULL
	result = con.Query("SELECT add_one(NULL)");
	REQUIRE(CHECK_COLUMN(result, 0, {Value()}));
	// NULL, 1, 2, 3 -> NULL, 2, 3, 4
	result = con.Query("SELECT add_one(i) FROM integers ORDER BY 1");
	REQUIRE(CHECK_COLUMN(result, 0, {Value(), 2, 3, 4}));
	// 2, 3 -> 3, 4
	result = con.Query("SELECT add_one(i) FROM integers WHERE i > 1 ORDER BY 1");
	REQUIRE(CHECK_COLUMN(result, 0, {3, 4}));
}
```

After rebuilding, we can run our test by running the `unittest` program: `build/debug/test/unittest "Test add one function"`. On Windows, we can run it by running the `unittest` program and adding `"Test add one function"` as the command line parameters. For now though, our function still does not exist and hence our tests fail with the following error message:

`Catalog: Function with name add_one does not exist!`

##### Implementation
Now that we have created our test, we can add our implementation. The implementation for functions lives in the `src/function/scalar` directory. In this case, we will place our function in the `math` subdirectory. Create the file `src/function/scalar/math/add_one.cpp` and add the following body of code:

```cpp
#include "duckdb/function/scalar/math_functions.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

using namespace duckdb;
using namespace std;

static void add_one_function(ExpressionExecutor &exec, Vector inputs[], index_t input_count, BoundFunctionExpression &expr,
                   Vector &result) {
	// initialize the result
	result.Initialize(TypeId::INTEGER);
	// now loop over the input vector
	VectorOperations::UnaryExec<int32_t, int32_t>(
		inputs[0], result, [&](int32_t input) {
		return input + 1;
	});
}

void AddOne::RegisterFunction(BuiltinFunctions &set) {
  // register the function
	set.AddFunction(ScalarFunction("add_one", { SQLType::INTEGER }, SQLType::INTEGER, add_one_function));
}
```

Also add the file `add_one.cpp` to `src/function/scalar/math/CMakeLists.txt`.

To complete our function definition, we need to add a few more lines of boilerplate. First, in the file `src/function/scalar/math_functions.cpp` add the following snippet of code in the `RegisterMathFunctions` function:

```cpp
Register<AddOne>();
```

Finally, in the file `src/include/duckdb/function/scalar/math_functions.hpp`, add the following snippet of code:
```cpp
struct AddOne {
	static void RegisterFunction(BuiltinFunctions &set);
};
```

After that our function should work! We can now test our function from within the shell our again by running our unittest program and verifying that it provides the correct result in all cases.

##### Assignment: Creating your own function
A list of functions from other systems that DuckDB is currently lacking can be found [here](https://github.com/cwida/duckdb/issues/193). If you implement the functions successfully, feel free to submit a pull request! 

The following functions are good starting points:

`RTRIM(VARCHAR) -> VARCHAR             [ MySQL  ]`
Remove spaces on right side of string

`REVERSE(VARCHAR) -> VARCHAR           [ MySQL  ]`
Reverse a string (mind the unicode!)

`REPEAT(VARCHAR, INTEGER) -> VARCHAR   [ MySQL  ]`
Repeat the specified string a number of times

Note that for functions that return a string, the string should be added to the string_heap inside the function by calling the `result.string_heap.AddString()` function prior to returning. For an example of this, see e.g. `src/function/scalar/string/substring.cpp:59`.
