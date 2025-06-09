+++
title = "Org-mode feature testing"
author = ["Eirik Haustveit"]
date = 2025-06-07T23:42:00+02:00
tags = ["tag2"]
categories = ["category1"]
draft = false
+++

This is the second org-mode test to see how various org-mode features looks with different Hugo templates.


## <span class="org-todo done DONE">DONE</span> Lists and checkboxes <span class="tag"><span class="_feature">@feature</span></span> {#lists-and-checkboxes}

-   [ ] Dette er ein test
-   [X] Dette er ein test
-   [X] Dette er ein test


## Pictures {#pictures}


## Text formatting {#text-formatting}

-   We can have **bold** text
-   Also _italic_
-   And `monospace`
-   For key bindings `C-c`
-   And when you ~~regret something~~
-   And when <span class="underline">someting is really important</span>


## Tables {#tables}

[org#Column formulas](https://orgmode.org/manual/Column-formulas.html "Emacs Lisp: (info \"(org) Column formulas\")")

<div class="table-caption">
  <span class="table-number">Table 1:</span>
  First demo table
</div>

| A  | B  | C  |
|----|----|----|
| 2  | 3  | 4  |
| 5  | 10 | 21 |
| 33 | 43 | 11 |
|    |    |    |

<div class="ox-hugo-table zebra-striping sane-table">
<div class="table-caption">
  <span class="table-number">Table 2:</span>
  Table with verbatim CSS
</div>

| a | b |
|---|---|
| b | s |
|   |   |

</div>


### Spreadsheet {#spreadsheet}

The tables in org-mode are a spreadsheet.

| Candidate | A  | B  | C   | D  | Mean  | Std. dev | RMS  |
|-----------|----|----|-----|----|-------|----------|------|
| 1         | 50 | 25 | 100 | 10 | 46.25 | 39.4     | 57.5 |
| 2         | 75 | 25 | 75  | 50 | 56.25 | 23.9     | 59.9 |
| 3         | 50 | 75 | 60  | 25 | 52.5  | 21.0     | 55.6 |


### Plotting {#plotting}

Org-mode supports plotting through gnuplot.

| Sede      | Max cites | H-index |
|-----------|-----------|---------|
| Chile     | 257.72    | 21.39   |
| Leeds     | 165.77    | 19.68   |
| Sao Paolo | 71.00     | 11.50   |
| Stockholm | 134.19    | 14.33   |
| Morelia   | 257.56    | 17.67   |

<a id="table--data-table"></a>

| x | y1 | y2 |
|---|----|----|
| 0 | 3  | 6  |
| 1 | 4  | 7  |
| 2 | 5  | 8  |


## Code listings {#code-listings}

This is a code listing for C:

```C
#include <stdio.h>

int main(){
  printf("Dette er ein test");
}
```


## PlantUML {#plantuml}

```c
!theme spacelab
a -> b
b -> c
```

```c
Bob -> Alice : Hello World!
```


## Citation {#citation}


## Math {#math}

Inline math \\(R = \frac{U}{I}\\)

\\[f(x) = \int\_{2}^{4} x^{4}+2x^{3}\\]

\begin{equation}
\label{eq:1}
C = W\log\_{2} (1+\mathrm{SNR})
\end{equation}
