# GNU tools for data analysts

## Introduction
The idea here is that the basic verbs of SQL/dplyr have analogues among the GNU
utilities, so we can leverage familiarity with data processing pipelines to
learn to use them. I'll use a pipe-delimited table to demonstrate. However, we
won't often need to use these tools with actual tables (I'd rather just read the
data into R and use `dplyr`). The real benefit is noticing that most
command-line utilities print lines of text to the screen that look enough like a
data.frame, so learning these little pipeline tools and tricks amplifies the
usefuleness of every other tool you learn -- you automatically have access to
this basic data analysis starter kit that you can pipe any output through. The
case study at the end shows how to use these tools to summarize the contents of
a nested directory structure.

The first example will use this pipe-delimited text file of the iris data:

```bash
@ butterfly (0): head -4 iris.csv

Sepal.Length|Sepal.Width|Petal.Length|Petal.Width|Species
5.1|3.5|1.4|0.2|setosa
4.9|3|1.4|0.2|setosa
4.7|3.2|1.3|0.2|setosa
```

To make the examples easier to follow, I'll sometimes pipe results through the
`column` command in order to better display the tabular structure of the data:

```bash
@ butterfly (0): head -4 iris.csv | column -s'|' -t

Sepal.Length  Sepal.Width  Petal.Length  Petal.Width  Species
5.1           3.5          1.4           0.2          setosa
4.9           3            1.4           0.2          setosa
4.7           3.2          1.3           0.2          setosa

```

To make it easier, I'll alias that last part:

```bash
alias prettify="column -s'|' -t"
```

## filter rows with `grep`

`grep` is a lot like `dplyr::filter`:

```bash
@ butterfly (0): grep 'virginica' iris.csv | head -3 | prettify

6.3  3.3  6    2.5  virginica
5.8  2.7  5.1  1.9  virginica
7.1  3    5.9  2.1  virginica
```

Note that `grep`, like all of the tools reviewed here, will take either a
filename (in which case it operates on the contents of the file) or the contents
of stdin as its input, e.g.:

```bash
@ butterfly (0): cat iris.csv | grep 'virginica' | head -3 | prettify

6.3  3.3  6    2.5  virginica
5.8  2.7  5.1  1.9  virginica
7.1  3    5.9  2.1  virginica
```

# select columns with `cut` + `rev`

`cut` is kind of `dplyr::select` -- it assumes that the text is delimited, and
takes as arguments the **d**elimiter and **f**ield (column) numbers of the
fields to select. So to select the second, fourth, and fifth columns, delimiting
by the pipe (`|`):

```bash
@ butterfly (0): cut iris.csv -d'|' -f2,4-5 | prettify | head -4

Sepal.Width  Petal.Width  Species
3.5          0.2          setosa
3            0.2          setosa
3.2          0.2          setosa
```

As we will see in the case study at the end, we don't always know how many
fields there will be in the input data. That's not a problem if we want to
select from the left of the table, but what if we want something like "the
second-to-last and last columns?" We can use the `cut | rev | cut` trick: `rev`
**rev**erses the input, for instance:

```bash
@ butterfly (0): echo 'hello world' | rev

dlrow olleh
```

So to select the last- and second-to-last columns, I can reverse the input,
select the first and second columns, and then reverse again so that everything
is right-side-forward:

```bash
@ butterfly (0): rev iris.csv | cut -d'|' -f1-2 | rev | prettify | head -4

Petal.Width  Species
0.2          setosa
0.2          setosa
0.2          setosa
```

# summarizing with `wc`

`wc` does some of what `dplyr::summarize` or `base::nrow` will do. `wc -l`
counts the number of **l**ines in the input, so to see the total number of rows:

```bash
@ butterfly (0): wc -l iris.csv # note: includes header
151 iris.csv
```

Or just the number of rows with species `versicolor`:

```bash
@ butterfly (0): grep 'versicolor' iris.csv | wc -l
50
```

# sorting and grouped summaries with `sort` + `uniq`

`sort` sorts its input, while `uniq` takes input that includes repeated lines,
and removes the repeats. By using them together, you de-duplicate the entire
input to see a distinct list:

```bash
@ butterfly (0): cut -d'|' -f5 iris.csv | sort | uniq

Species
setosa
versicolor
virginica
```

Use the `-c` switch to add a **c**ount of times that each unique value occurred, as
in `dplyr::group_by` + `dplyr::summarize`:

```bash
@ butterfly (0): cut -d'|' -f5 iris.csv | sort | uniq -c

      1 Species
     50 setosa
     50 versicolor
     50 virginica
```

# mutating with `tr`/`awk`/`sed`/etc.

There are a number of tools that take lines of input and transform them in some
way, such as replacing patterns. These are sort of like `dplyr::mutate`. Here we
take the previous summary and convert everything to upper-case:

```bash
@ butterfly (0): cut -d'|' -f5 iris.csv \
    | sort | uniq -c \
    | tr '[:lower:]' '[:upper:]'

      1 SPECIES
     50 SETOSA
     50 VERSICOLOR
     50 VIRGINICA
```

# It's all delimited lines of text

As mentioned above, I don't actually use these command-line tools to work with
tables like this. But most command-line tools print out lines of text, often
with some kind of structure using a consistent delimiter, to stdout. So we
can pipe outputs of all sorts of tools through these utilities. For instance, if
my browser is hanging and I want to find its process id in order to kill it:

```bash
@ butterfly (0): ps -A | grep firefox

  655 ??        86:18.88 /Applications/Firefox.app/Contents/MacOS/firefox
77413 ttys011    0:00.00 grep firefox
```

# Case study: nested directory structure

We sometimes receive entire directories of data from partners, where the data
files are nested within varying levels of the structure. Sometimes, the nested
directory structure itself encodes some useful bit of metadata. `proj`, for
instance, has a bunch of data files (stored as *.txt files) within a nested
directory structure that sort of encodes the date:

```bash
@ butterfly (0): tree proj | head -15

proj
├── 2018
│   ├── Q1
│   │   ├── 00.txt
│   │   ├── 01.txt
│   │   ├── 02.txt
│   │   ├── 03.txt
│   │   ├── 04.txt
│   │   ├── 05.txt
│   │   ├── 06.txt
│   │   ├── 07.txt
│   │   ├── 08.txt
│   │   └── 09.txt
│   ├── Q2
│   │   ├── 01.txt

```

I say "sort of" because the files are stored in various ad-hoc sub-directories,
so that all of the files are not found at the same level down the hierarchy. By
filtering out the ".txt" lines from tree, I can see the full directory
structure:

```bash
@ butterfly (0): tree proj | grep -v '\.txt'

proj
├── 2018
│   ├── Q1
│   ├── Q2
│   ├── Q3
│   └── Q4
├── 2019
│   ├── Q1
│   ├── Q2
│   ├── Q3
│   └── Q4
└── new-batch
    └── files-from-xxx
        ├── 2016
        │   ├── Q1
        │   ├── Q2
        │   ├── Q3
        │   └── Q4
        └── 2017
            ├── Q1
            ├── Q2
            ├── Q3
            └── Q4

22 directories, 147 files
```

## `find`

The `find` tool looks recursively through directories for files or directories
matching some pattern(s). It's really powerful, `man find` will come in handy.
For now, the most useful options are:

- `name`, `iname`: search for *filenames* matching a given pattern. `iname`
  means the search will be case-insensitive, which is what I usually want
- `path`, `ipath`: search for *paths* matching a given pattern.
- `type`: whether you want to look for regular files, directories, symlinks,
  etc.

So, for instance I can view the full path of every data file (I'm using `shuf`
here to **shuf**fle the input, so that we can see a random selection of results,
rather than just the first few in order):

```bash
@ butterfly (0): find proj -iname '*.txt' | shuf | head -7

proj/new-batch/files-from-xxx/2017/Q2/file.txt
proj/2019/Q3/04.txt
proj/new-batch/files-from-xxx/2017/Q4/16.txt
proj/2019/Q2/02.txt
proj/new-batch/files-from-xxx/2017/Q4/18.txt
proj/new-batch/files-from-xxx/2016/Q3/04.txt
proj/new-batch/files-from-xxx/2017/Q4/13.txt
```

*Note: we often have some processing code that we want to run on every input data
file. It is much easier to use `find`, as above, to generate the list of
filenames for the processing code to run on, rather than writing some custom
recursive code in R or Python to walk the directory structure.*

## A ragged array

Notice that we can treat the path-delimiter `/` as a field delimiter, and by
doing so see that `find` is returning something resembling the tables we were
working with earlier:

```bash
@ butterfly (0): find proj -iname '*.txt' | shuf | head -7 | column -s'/' -t

proj  2019       Q3              05.txt
proj  new-batch  files-from-xxx  2017    Q1  10.txt
proj  2018       Q1              05.txt
proj  new-batch  files-from-xxx  2016    Q4  01.txt
proj  new-batch  files-from-xxx  2016    Q2  00.txt
proj  new-batch  files-from-xxx  2017    Q4  11.txt
proj  new-batch  files-from-xxx  2017    Q4  12.txt
```

It's not quite a table, since different rows can have different numbers of
fields. But notice also that the date metadata is encoded in the second- and
third-to-last columns of each row:

```bash
@ butterfly (0): find proj -iname '*.txt' \
    | shuf \
    | rev | cut -d'/' -f2-3 | rev \
    | head -7

2017/Q1
2016/Q2
2016/Q4
2019/Q3
2018/Q3
2018/Q1
2016/Q4
```

So, we have a way of reporting the distribution of data by date:

```bash
@ butterfly (0): find proj -iname '*.txt' \
    | rev | cut -d'/' -f2-3 | rev \
    | sort | uniq -c \
    | tr '/' '-' \
    | sort -r

     31 2017-Q4
     20 2019-Q4
     17 2016-Q2
     11 2017-Q3
     10 2018-Q1
     10 2017-Q1
      9 2016-Q3
      7 2018-Q3
      7 2018-Q2
      6 2019-Q2
      5 2019-Q3
      5 2016-Q4
      4 2017-Q2
      3 2019-Q1
      1 2018-Q4
      1 2016-Q1
```

Since our projects always keep source code in a directory named "src," I can use
the same trick to find out what the most common programming languages are based
on number of scripts. Note here I use the dot as the field delimiter, because I
want to isolate the file extensions:

```
@ butterfly (0): find ~/git -ipath '*/src/*.*' \
    | rev | cut -d'.' -f1 | rev \
    | tr '[:upper:]' '[:lower:]' \
    | sort | uniq -c \
    | sort -r

    952 py
    478 pyc
    459 r
    142 xml
    102 pdf
     73 json
     53 java
     52 go
     19 rnw
     18 tex
     14 csv
     11 md
      9 rmd
      6 jl
      5 ipynb
      4 sh
      2 sty
      2 rdata
      2 html
      2 docx
      2 c
      2 bst
      1 rvim
      1 rb
      1 mmd
      1 mk
      1 make
      1 jar
      1 ipynb_checkpoints
      1 history
      1 depricated
      1 bib
```
