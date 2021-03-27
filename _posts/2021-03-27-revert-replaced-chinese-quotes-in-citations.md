---
layout: post
title: Revert replaced chinese quotes in citations
date: 2021-03-27
author: Fxzjshm
category: Other
tags: [pandoc, csl]
---

For some unknown reason, the settings of quotes in [jgm/citeproc/locales/zh-CN.xml](https://github.com/jgm/citeproc/blob/master/locales/zh-CN.xml) and (originally) [citation-style-language/locales/locales-zh-CN.xml](https://github.com/citation-style-language/locales/blob/master/locales-zh-CN.xml) is:
``` xml
    <!-- PUNCTUATION -->
    <term name="open-quote">《</term>
    <term name="close-quote">》</term>
    <term name="open-inner-quote">〈</term>
    <term name="close-inner-quote">〉</term>
    <term name="page-range-delimiter">–</term>
```
which will convert `“”` into `《》` in titles of the citation, e.g. `“软色情”，正在榨干这一代的年轻人` -> `《软色情》，正在榨干这一代的年轻人`.

Temporary fix: add
```markdown
lang: en
```
in the metadata of markdown files, i.e.
```markdown
---
lang: en
mainfont: simsun.ttc
bibliography: [ref.bib]
---

# Heading 1
...
```
