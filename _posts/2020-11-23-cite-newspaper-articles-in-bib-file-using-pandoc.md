---
layout: post
title: Cite newspaper articles in .bib file using pandoc
date: 2020-11-23
author: Fxzjshm
category: Other
tags: [pandoc, bibtex]
---

Use the `entrysubtype={newspaper}` property to indicate that this is an article in newspaper, or `magazine` for those in magazines.  

Exmaple:  
```
@article{test,
entrysubtype={newspaper},
title={test-title},
journal={test-newspaper},
author={L, F M},
year={0000}, 
month={Jan},
pages={2–3}
}

```

Reference: <https://github.com/jgm/pandoc/blob/fec8223d3a6979f675513300b2211c6235b8bccd/src/Text/Pandoc/Citeproc/BibTeX.hs#L1071>
