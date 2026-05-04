---
title: Code for creating the coil
authors:
  - name: Leeor Alon, Sipan Hovsepian
kernelspec:
  name: python3
  display_name: Python 3
exports:
  - format: typst
    template: lapreprint-typst
    output: exports/coil.pdf
---


## Problem Statement


In an ideal Vector Network Analyzer (VNA) measurement, the Device Under Test (DUT) is connected directly to the VNA ports to minimize signal attenuation and phase errors. However, physical constraints often make direct connection impossible.



```{code-cell} ipython3
import matplotlib.pyplot as plt
import numpy as np
from numpy.linalg import norm
import magpylib as magpy
import pickle
import pyvista as pv
import pandas as pd
```

