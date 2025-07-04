---
layout: post
title: Set up sudo mail warn
date: 2025-07-04
author: Fxzjshm
category: Linux
tags: [Linux]
---

Typst 目前并不支持 PostScript 甚至 PDF, 若文档需要插入 PGPLOT 绘制的 .ps 图, 需要转换. 以下探索了一种 *看起来* **无损** 转换为 SVG 的方法.  
Typst does not support PostScript and even PDF for now, if need insert .ps figure plotted by PGPLOT in document, it nees convert. Here discovered a method to convert PS to SVG that *seems* **losslessly**.

```bash
for f in *.ps; do 
  eps2eps $f $f.eps;
  gs -dBATCH -dEPSFitPage -dAutoFilterColorImages=false -dColorImageFilter=/FlateEncode -dNOPAUSE -sDEVICE=pdfwrite -sOutputFile=$f.eps.pdf $f.eps
  qpdf $f.eps.pdf  --rotate=180 -- $f.eps.rotated.pdf
  inkscape --export-type=svg --export-area-drawing $f.eps.rotated.pdf
  rm $f.eps $f.eps.pdf $f.eps.rotated.pdf
done
```

注意很多其它方法会导致内部位图被降采样.  
Notice that many other methods may downsample internal rastered image.

注意对不同软件绘制的图, 需要旋转的度数不同 (例如 PulsarX 180 度, PRESTO 与 DSPSR/PSRCHIVE 90 度).  
Notice for figure plotted by different software, degrees of rotation is different (e.g. 180 deg for PulsarX, 90 deg for PRESTO and DSPSR/PSRCHIVE).
