---
jupytext:
    text_representation:
        extension: .md
        format_name: myst
        format_version: 0.13
        jupytext_version: 1.19.1
kernelspec:
    display_name: geo
    language: python
    name: python3
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

