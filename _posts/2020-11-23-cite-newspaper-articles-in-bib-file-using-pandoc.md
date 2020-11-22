---
layout: post
title: Cite newspaper articles in .bib file using pandoc
date: 2020-11-23
author: Fxzjshm
category: Other
tags: [pandoc, bib]
---

Use the `entrysubtype` property to indicate that this is an article in newspaper.  

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

Reference: <https://github.com/jgm/pandoc/blob/master/src/Text/Pandoc/Citeproc/BibTeX.hs>
