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


## Tables {#tables}

[org#Column formulas](https://orgmode.org/manual/Column-formulas.html "Emacs Lisp: (info \"(org) Column formulas\")")

| A  | B  | c  |
|----|----|----|
| 2  | 3  | 4  |
| 5  | 10 | 21 |
| 33 | 43 | 11 |
|    |    |    |


### Spreadsheet {#spreadsheet}


### Plotting {#plotting}


## Code listings {#code-listings}

```C
int main(){
  printf("test");
}
```


## Citation {#citation}


## Math {#math}

Inline math \\(R = \frac{U}{I}\\)

\\[f(x) = \int\_{2}^{4} x^{4}+2x^{3}\\]

\begin{equation}
\label{eq:1}
C = W\log\_{2} (1+\mathrm{SNR})
\end{equation}
