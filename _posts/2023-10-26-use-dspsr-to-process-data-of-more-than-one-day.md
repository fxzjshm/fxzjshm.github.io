---
layout: post
title: Use DSPSR to process data of more than one day
date: 2023-10-26
author: Fxzjshm
category: Pulsar
tags: [Linux]
---

When processing continuous observation of more than 1 day,
an error occurred from DSPSR:
```
Error::message
        epoch 60230.28412758417979894210 not spanned by ChebyModelSet
```

The error is related to TEMPO2 wrapper `T2Predictor`, where TEMPO2 complains `ChebyModelSet_OutOfRange`.
This points to the command line of DSPSR calling TEMPO2, which does have a span of one day:
```
tempo2 -npsr 1 -f pulsar.par -pred "fast 60229.283869345 60230.283869345 200.006942749023438 350.006942749023438 12 2 3599.9999999999998" > stdout.txt 2> stderr.txt
```

Further searching shows the time span comes from `dspsr/Signal/Pulsar/Fold.C`, line 243.
Changed this magic number
```diff
diff --git a/Signal/Pulsar/Fold.C b/Signal/Pulsar/Fold.C
index 009c1615..0c5c42f9 100644
--- a/Signal/Pulsar/Fold.C
+++ b/Signal/Pulsar/Fold.C
@@ -262,7 +262,7 @@ dsp::Fold::get_folding_predictor (const Pulsar::Parameters* params,
  /*
   * Tempo2 predictor code:
   *
   * Here we make a predictor valid for the next 24 hours
   * I consider this to be a bit of a hack, since theoreticaly
   * observations could be longer, and it's a bit silly to make
   * such a polyco for a 10 min obs.
   *
   */
-  MJD endtime = time + 86400;
+  MJD endtime = time + 10*86400;
 
   generator->set_site( observation->get_telescope() );
   generator->set_parameters( params );
   generator->set_time_span( time, endtime );
```
so folding can be done further.

<!-- more -->
