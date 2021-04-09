Large Data in R: Or How I Learned to Stop Worrying and Allocate That
Vector
================
Ben Sabath

Updated April 09, 2021

------------------------------------------------------------------------

## Table of Contents

-   [Part 0: Environment Set Up](#part-0-environment-set-up)
-   [Part 1: What is Large Data?](#part-1-what-is-large-data)
-   [Part 2: How Does R Work With
    Memory?](#part-2-how-does-r-work-with-memory)
-   [Part 3: Thinking about Memory Challenges in
    R](#part-3-thinking-about-memory-challenges-in-r)
-   [Part 4: Working With Data that “Mostly
    Fits”](#part-4-working-with-data-that-mostly-fits)
-   [Part 5: Working With Data That “Doesn’t
    Fit”](#part-5-working-with-data-that-doesnt-fit)
-   [Additional Resources](#additional-resources)

------------------------------------------------------------------------

## Part 0: Environment Set Up

**PLEASE DO THIS BEFORE THE WORKSHOP**

There are multiple ways to set up your environment for this course. Our
focus will be more on the concepts underlying Large Data in R and how to
work through problems, rather then executing specific blocks of code.
However if you’d like to follow along and ensure that you have all
packages you need installed, I’ve provided a `conda` environment in this
repo for your use.

Note: This course assumes that you are comfortable using the command
line and are working on a Unix based system (MacOS or Linux). The
assistance I’ll be able to provide for attendees with windows computers
will be limited.

#### STEP 0 - Clone This Repo

If you’re reading this, you’ve found the git repository with materials
for this course. The easiest way to download all of the materials and
have them properly arranged is to clone this repo. To do this run the
following command

    git clone git@github.com:mbsabath/large_data_in_R

This will create a local copy of this repo on your computer. `cd` to
that directory for the rest of this setup.

#### STEP 1 - Install Conda

If you’re working on a computer that doesn’t currently have conda
installed, you can install miniconda [using this
link](https://docs.conda.io/en/latest/miniconda.html). I recommend the
Python 3.9 version.

#### STEP 2 - Create The Environment

Included in the repo is the file `large_data_env.yml`. This file lists
the packages needed for this course. Conda is great for environment
management and environment sharing since it handles installing all of
the dependencies needed, and can support set up on multiple operating
systems. Creating conda environments for your projects is a separate
subject, but is a great way to make your research projects easy for
others to use and to support reproducibility. To install this
environment run:

    conda env create -f large_data_env.yml

You will be prompted to download and install a number of packages,
please install things as prompted.

If everything worked, you should see an environment named
`large_data_in_R` listed when you run

    conda env list

#### STEP 3 - Run R and Install One Additional Package

To activate the environment, run the following command:

    source activate large_data_in_R

If this is successful, your terminal prompt will change to look
something like this:

    (large_data_in_R) <username> large_data_in_R %

To run Rstudio using the environment, it’s important to run it from the
terminal. To start Rstudio from the terminal, enter

    rstudio

from the terminal where you’ve activated the environment. The rstudio
window that opens will have all required packages already installed,
with the exception of the `chunked` package, which is not available
through conda. To install that package, please run:

``` r
install.packages("chunked")
```

Once `chunked` is installed, your environment is good to go!

## Part 1: What Is Large Data?

### 1.1: Defining Big Data

There is often discussion in data science circles about “what counts” as
big data. For some, it’s data that can’t fit in memory, for others it’s
data so big, it can’t fit on a single hard drive. It might be one single
large file, or TBs upon TBs of small files. It might just be an excel
file large enough that Excel complains when you use it.

![Qui Gon Jinn](images/Qui-Gon-Jinn_d89416e8.jpeg) *There’s always
bigger data - Qui Gon Jinn*

For me, I find the most useful definition to be “Data that is big enough
that you cannot use standard workflows”. Any time data is big enough
that you need to adjust your processes to account for that, you’re
entering the world of “Big Data”.

That said, for R users, and generally most academic users, the main big
data issue is data that is too big to be easily worked with in memory.
That is the problem this workshop will seek to address.

### 1.2 What is Data?

#### 1.2.1 Data on Disk

When thinking about and working with data, it’s important to understand
the layers of abstraction present and how they interact with each other.
Think of an html file. On disk, the file is just a structured section of
bits. However, depending on how its encoded (either ASCII or UTF-8), we
have instructions for how to interpret those bits as a text file. An
html file in text format is more readable than the raw bits and can be
edited by a knowledgeable web developer; however, it is not until the
final layer of abstraction, when the information contained in the page
is rendered by your browser that the information in the file can be
easily interpreted.

![Data Abstraction Layers](images/data_abstraction.png) *Examples of
layers of abstraction in data*

The same process occurs with any data we want to analyze. Whatever its
ultimate structure or purpose is, all data is blocks of bits, with
abstraction layers that define how those bits should be interpreted.

CSVs, TSVs, and Fixed width files are all text based formats that are
typically used to define data frame like data. They are useful as they
are both human and machine readable, but text is a relatively
inefficient way to store data.

There are also a number of binary formats (which often also have some
compression applied on disk), such as `.fst` files, `hdf5` files, or R
specific `.RDS` files. These have the advantage of being more space
efficient on disk and faster to read, but this comes at the cost of
human readability.

#### 1.2.2 Data in Memory

*Note: this section is an over simplification*

This will be discussed more in part 2, but when working with large data
it’s important to have a sense of what computer memory is. We can think
of computer memory as a long tape. When data is loaded in to memory and
assigned to a variable, such as when you run a line like

``` r
x <- 1:100
```

the operating system finds a section of memory large enough to contain
that object (in this case 100 integers), reserves it, and then connects
its location in memory to the variable. Some objects need to all be on a
contiguous section of the tape, some objects can be spread across
multiple parts of it. Regardless, the program needs to know interpret
the bits stored in memory as a given object. How much space objects take
up depends on the system, but the classic base objects in C are as
follows:

-   integer: 32 or 64 bits, depending on architecture
-   double: floating point, typically 64 bits
-   char: typically 8 bits
-   boolean: In theory could be 1 bit, in practice varies by language
    and processor

One common type not seen here is the string. strings are arrays of
characters stored in memory. The characters need to be in a contiguous
section of memory, otherwise the system will not be able to interpret
them as a single string.

### 1.3 Flow of Data in a Program

![Flow of Data in A Program](images/Computer%20Data%20Flow.png) In a
typical program, we start with data on disk, in some format. We read it
in to memory, do some stuff to it on the CPU, store the results of that
stuff back in memory, then write those results back to disk so they can
be available for the future.

There are limits on this. A single CPU can only do one calculation at a
time, so if we have a lot of calculations to do, things may take a
while. Similarly, memory can only hold so much data, and if we go over
that limit, we either get an error, or have to find some work around.
Understanding this flow is key to analyzing how to find those
workarounds, and what the limits on our workarounds are.

## Part 2: How Does R Work With Memory?

Basically everything in this section is pulled from Chapters 2 and 3 of
[Hadley Wickham’s Advanced
R](https://adv-r.hadley.nz/names-values.html). I can’t recommend that
enough as a resource to really understand what’s going on in R under the
covers.

### 2.1 R as a Language

Why do people use R? There are a number of other statistical programming
languages out there.

``` r
library(lobstr)
```

### 2.2 Variable Assignment and Copy behavior

### 2.3 Data types and variable size

### 2.4

## Part 3: Thinking about Memory Challenges in R

### 3.1 Trade Offs When Working With Big Data

If you’re working with data large enough to hit the dreaded
`cannot allocate vector` error when running your code, you’ve got a
problem. When using your naive workflow, you’re trying to fit too large
an object through the limit of your system’s memory.

![The Evergiven (please put her back)](images/big%20boat.jpg) *An
example of a large object causing a bottleneck*

When you hit a memory bottleneck (assuming it’s not caused by a simple
to fix bug in your code), there is no magic quick solution to getting
around the issue. There’s a number of resources that need to be
considered when developing any solution:

-   Development Time
-   Program Run Time
-   Computational Resources
    -   Number of Nodes
    -   Size of Nodes

A simple solution to a large data problem is just to throw more memory
at it. Depending on the problem, how often something needs to be run,
how long it’s expected to take, etc, that’s often a decent solution, and
minimizes the development time needed. However, compute time could still
be extended, and the resources required to run the program are quite
expensive. For reference, a single node with 8 cores and 100GB of memory
costs 300$/month on Amazon. If you need 400GB of memory (which some
naive approaches using the CMS data I’m familiar with do), the cost for
a single node goes up to 1200 per month.

Conversely, since all computing processes can be theoretically
parallelized (although just because something is parallelized, it
doesn’t mean it will perform better), with enough development time, any
algorithm could be made to run in a standard personal computer’s memory.

The appropriate solution depends on the needs and constraints of the
specific project.

### 3.2 Types of Memory Challenges

For the remainder of this workshop, we’re going to cover two types of
memory problems. The separation between these is fairly arbitrary, and
in most cases you’ll want to use techniques from both sets of approaches
to resolve your big data problems.

The two types are as follows: - Data that “mostly fits” in memory. This
is data where you can read the entire object in to memory, and even run
some basic calculations on the set. However complicated calculations or
data cleaning operations lead to you running out of memory. - Data that
doesn’t fit in memory. This is data where you can’t even load what you
need in to memory

## Part 4: Working With Data that “Mostly Fits”: Optimizing Memory Usage

### 4.1 General Strategies

When working with data that “mostly fits”, our main goals our to reduce
the memory impact of all the computational processes being done.

#### 4.1.1 In Place Changes

As previously discussed, R has unpredictable behavior with regard to
data duplication. If a vector has been referred to at least twice at any
point in the code, any change to that vector will lead to duplication of
the entire vector. When we’re dealing with data approaching the size of
memory, any duplication poses the risk of creating an error and halting
our program.

Working with In Place changes is the best way to avoid this duplication.
R List objects don’t require contiguous blocks of memory and allow for
in place changes. However, many of the default functions in R do not
support vectorized operations on list objects. Additionally, since the
standard Data Frame object in R is a list of vectors, we are unable to
do in place data manipulation with the tools in base R.

It is worth mentioning that pandas data tables in python, the standard
method for python data analysis have default support for in place
operations. While R is the standard tool in many academic disciplines,
Python is a strong alternative approach for many applications.

### 4.2 Inplace Data Frame Manipulation with `data.table`

One of the best tools out there for working with large data frames in R
is the `data.table` package. It implements the `data.table` object, with
many routines written and compiled in C++ and designed for parallel
processing. It also supports in place mutation operations on
data.frames, as well as having a nice syntax for data aggregation and
data joining.

To demonstrate this, lets create two reasonably sized random data
frames, one using a standard R data frame, and one using `data.table`.

``` r
set.seed(111917)
library(data.table)

df <- data.frame(x = rnorm(1e6), y = rnorm(1e6), z = rep_len(c("a", "b"), 1e6))
dt <- data.table(x = rnorm(1e6), y = rnorm(1e6), z = rep_len(c("a", "b"), 1e6))
```

Show data duplication within df

``` r
tracemem(df)
```

    ## [1] "<0x7f885d6880c8>"

``` r
df$x <- df$x + 1
```

    ## tracemem[0x7f885d6880c8 -> 0x7f885f080328]: eval eval withVisible withCallingHandlers handle timing_fn evaluate_call <Anonymous> evaluate in_dir block_exec call_block process_group.block process_group withCallingHandlers process_file <Anonymous> <Anonymous> 
    ## tracemem[0x7f885f080328 -> 0x7f885f0803c8]: $<-.data.frame $<- eval eval withVisible withCallingHandlers handle timing_fn evaluate_call <Anonymous> evaluate in_dir block_exec call_block process_group.block process_group withCallingHandlers process_file <Anonymous> <Anonymous> 
    ## tracemem[0x7f885f0803c8 -> 0x7f885f0804b8]: $<-.data.frame $<- eval eval withVisible withCallingHandlers handle timing_fn evaluate_call <Anonymous> evaluate in_dir block_exec call_block process_group.block process_group withCallingHandlers process_file <Anonymous> <Anonymous>

``` r
untracemem(df)
```

No duplication with `data.table`

``` r
tracemem(dt)
```

    ## [1] "<0x7f886301b400>"

``` r
dt[, x := x+1]
untracemem(dt)
```

In place memory work also improves computational speed and efficiency.
There are a number of reasons that `data.table` is faster than the
standard R implementation, but a large part of that is that R no longer
needs to wait for the system to allocate additional memory for each of
these calculations.

In general, R duplication is somewhat unpredictable, as it depends on
the number of times an object has been referenced at any point during a
session. `tracemem` is a necessity for identifying duplication events.
However for complex calculations that can’t be implemented using
`data.table` or other packages specifically built for in place
calculations, you may need to implement your algorithm using `rcpp` or
with the R API in C.

### 4.3 Avoid Keeping Uneeded Data in Memory

Frequently, data cleaning operations contain mutation, filtering, and
aggregation steps. When there is no risk of filling memory, the order
these steps are performed doesn’t matter. However, when memory usage
needs to be taken in to consideration, we no longer have the same
flexibility in our approaches.

### 4.4 Manage Data Types

As previously mentioned, R vectors contain many atomic values, each of
which is ultimately one of the base data types in R. These atomic data
types have varying sizes. While the size of these atomic objects differs
only on the level of bytes, when multiplied millions or billions of
times, these small differences add up.

Let’s start by comparing `interger` and `float` vectors:

``` r
int_vec <- rep_len(2L, 1e6)
float_vec <- rep_len(2, 1e6)
object.size(int_vec)
```

    ## 4000048 bytes

``` r
object.size(float_vec)
```

    ## 8000048 bytes

Here we see that the integer vector, specified by using the `L`
following the number is half the size of the float vector, which was
created with syntax more like standard R.

However, memory usage based on data type is not always predictable. We
can see this when looking at boolean variables. Theoretically, a boolean
should be representable with a single bit, and maybe two bits if we need
to account for `NA` values. Let’s see what actually happens:

``` r
int_vec <- rep_len(c(0L, 1L), 1e6)
bool_vec <- rep_len(c(T, F), 1e6)
object.size(int_vec)
```

    ## 4000048 bytes

``` r
object.size(bool_vec)
```

    ## 4000048 bytes

Boolean vectors and integer vectors are the exact same size. This is
because R stores boolean vectors as 32 bits, even though most bits
aren’t needed logically.

String vectors are also an interesting case. Keeping in mind that each
character in a string is one byte, we’d expect a vector of repeated long
strings to be significantly larger than an equivalent integer vector:

``` r
int_vec <- 1:1e6
str_vec <- rep_len(c("Really long string number one",
                     "Second long string, that is even longer than the first"),
                   1e6)
int_str_vec <- as.character(int_vec)
object.size(int_vec)
```

    ## 4000048 bytes

``` r
object.size(str_vec)
```

    ## 8000240 bytes

``` r
object.size(int_str_vec)
```

    ## 64000048 bytes

The reason for this difference is that in order to optimize memory
usage, R sets up a “string pool”. where it stores all unique strings.
Vectors of strings are then just pointers to objects in that shared
string pool. This is why the string representation of all numbers
1-1,000,000 is much larger than a vector of repeated long strings. This
also reduces the need for contiguous memory when dealing with strings,
as we only need contiguous memory for the pointers, and not for the
strings themselves.

## Part 5: Working With Data That “Doesn’t Fit”

## Additional Resources

-   [Hadley Wickham’s Advanced R](https://adv-r.hadley.nz/index.html)
-   [Data.Table Syntax Cheat
    Sheet](https://www.datacamp.com/community/tutorials/data-table-cheat-sheet)