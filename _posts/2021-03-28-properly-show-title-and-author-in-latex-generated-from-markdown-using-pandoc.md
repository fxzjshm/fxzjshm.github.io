---
layout: post
title: Properly show title and author in latex generated from markdown using pandoc
date: 2021-03-28
author: Fxzjshm
category: Other
tags: [pandoc, latex]
---

`template.tex`:
```latex
\begin{document}

\section{$title$}

\begin{center}
$author$
\end{center}


$body$

\end{document}
```
(I use `\section{}` for title because it corresponds to `#` in markdown.)

Markdown metadata:
```yaml
---
title: 'The Title'
author:
  - 'Author&nbsp;1'
---
```

Reference: <https://pandoc.org/MANUAL.html#metadata-variables>

<!-- more -->
