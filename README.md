a lot of the programs here operate on stdin, but also have an argument for a filename, so you can specify a filename rather than stdin. the fact that the programs can read input from stdin, and write their output to stdout, makes it easy to chain commands together in pipelines. as a result, some of these commands naturally complement each other, and i'll highlight common pairs as i go.

# viewing files

## `cat` and `zcat`

prints all lines of a file (or multiple files) to stdout. `zcat` is particularly useful because it can print compressed files. in conjunction with the pipe, that means you can process compressed files without having to separately unzip them:

```bash
@ butterfly (0): zcat iris.csv.zip | head -3

Sepal.Length|Sepal.Width|Petal.Length|Petal.Width|Species
5.1|3.5|1.4|0.2|setosa
4.9|3|1.4|0.2|setosa
```

## `head` and `tail`
print the beginning (`head`) or end (`tail`) of input. specify the number of lines using the `-n` option, where *n* is the number of lines.

## pagers: `more`, `less`, etc.

pagers view files a "page" at a time. i use `less`, which allows you to scroll up and down through a file, and search using the `/` command (like in vim). the advantage here is that you don't load an entire file all at once, just the lines that are displayed. this can be useful to preview the contents of a large file, but i also use it in pipelines to preview the output of a command that i'm not sure about. for instance, i recently wanted to look for all R files within my projects directory (see `find` below), but i wanted to make sure i was writing the command correctly before outputting the results to a file, so previewed the command using:

```bash
find ~/git -iname '*.R' | less
```

# filter rows with `grep`

```bash
@ butterfly (0): grep 'virginica' iris.csv | head -4

6.3|3.3|6|2.5|virginica
5.8|2.7|5.1|1.9|virginica
7.1|3|5.9|2.1|virginica
6.3|2.9|5.6|1.8|virginica
```

# select columns with `cut` + `rev`

```bash
@ butterfly (0): cut iris.csv -d'|' -f2,4-5 | head -4

Sepal.Width|Petal.Width|Species
3.5|0.2|setosa
3|0.2|setosa
3.2|0.2|setosa
```

In order to select rows not based on counting from the left, but from the right (e.g. last column, second-to-last, etc) use `rev` to reverse the input, making the last column into the first:

```bash
@ butterfly (0): rev iris.csv | cut -d'|' -f1-2 | rev | head -4

Petal.Width|Species
0.2|setosa
0.2|setosa
0.2|setosa
```

# summarizing with `wc`

```bash
@ butterfly (0): wc -l iris.csv # note: includes header
151 iris.csv

@ butterfly (0): grep 'versicolor' iris.csv | wc -l
50
```

# sorting and grouped summaries with `sort` + `uniq`

```bash
@ butterfly (0): cut -d'|' -f5 iris.csv | sort | uniq -c

      1 Species
     50 setosa
     50 versicolor
     50 virginica
```

# mutating with `tr`/`awk`/`sed`/etc.

```bash
@ butterfly (0): cut -d'|' -f5 iris.csv \
    | sort | uniq -c \
    | tr '[:lower:]' '[:upper:]'

      1 SPECIES
     50 SETOSA
     50 VERSICOLOR
     50 VIRGINICA

```

# finding/searching

## `grep`

`grep` is a useful tool for searching through the contents of files. for instance:

```bash
@ butterfly (0): grep 'virginica' iris.csv | head -4

6.3|3.3|6|2.5|virginica
5.8|2.7|5.1|1.9|virginica
7.1|3|5.9|2.1|virginica
6.3|2.9|5.6|1.8|virginica
```

but it's helpful to think of `grep` not as something that only searches through files, but rather as something that searches through whatever you feed into stdin. for instance, to view a list of all dotfiles in my home directory, i can do:

```bash
ls -a ~ | grep '^\.'
```

`ls -a ~` lists all files (including those that begin with a dot, since i used the `-a` switch) in my home directory, and then `grep '^\.'` searches for lines with a dot at the beginning of the line.

### `grep` example: visualizing project structure

We sometimes receive entire directories of data from clients, where the data files are nested within varying levels of the structure. i often use `tree` to visualize a project's contents:

```
@ butterfly (0): tree proj | head -25

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
│   │   ├── 02.txt
│   │   ├── 03.txt
│   │   ├── 04.txt
│   │   ├── 05.txt
│   │   ├── 06.txt
│   │   └── 07.txt
│   ├── Q3
│   │   ├── 01.txt
│   │   ├── 02.txt
│   │   ├── 03.txt
```

In some cases, including this one, the directory structure itself encodes some meaningful bit of metadata. `tree` has the `-L` option to limit the number of levels down a tree you want to print, so if I wanted to see all the full date range for the data we received, I might start by trying:

```
@ butterfly (0): tree proj -L 2

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

12 directories, 0 files
```
But in this case the project has different levels of meaningful hieararchy in different places:

```
@ butterfly (0): tree -L 2 proj/new-batch/files-from-xxx/

proj/new-batch/files-from-xxx/
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

10 directories, 0 files
```

I can pipe the output from `tree` through `grep` to weed out the individaul $.txt$ files, and just look at the directory structure. the `-v` option makes `grep` work in opposite, it only returns the lines that *do not* match the search pattern:

```
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

## `grep -r`

A common need is to find some bit of text within all files in a project. For instance, you want to find every time you used a particular function within 

```
@ butterfly (0): find . -iname '*.txt' | rev | cut -d'/' -f2-3 | rev | sort | uniq -c
      1 2016/Q1
     17 2016/Q2
      9 2016/Q3
      5 2016/Q4
     10 2017/Q1
      4 2017/Q2
     11 2017/Q3
     31 2017/Q4
     10 2018/Q1
      7 2018/Q2
      7 2018/Q3
      1 2018/Q4
      3 2019/Q1
      6 2019/Q2
      5 2019/Q3
     20 2019/Q4
```

```bash
# 1. list full paths of .txt files
# 2. take the parent + grandparent dir names
# 3. count how many files in each

find . -iname '*.txt' \
    | rev | cut -d'/' -f2-3 | rev \
    | sort | uniq -c
```

```
@ butterfly (0): find ~/git -ipath '*/src/*.*' \
    | rev | cut -d'.' -f1 | rev \
    | tr '[:upper:]' '[:lower:]' \
    | sort | uniq -c \
    | sort -r

    950 py
    478 pyc
    454 r
    142 xml
    102 pdf
     73 json
     53 java
     52 go
     19 rnw
     18 tex
     14 csv
     11 md
      7 rmd
      6 jl
      4 sh
      4 ipynb
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

```bash
zcat iris.csv.zip | cut -d'|' -f5 | sort | uniq -c
```
