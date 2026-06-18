---
name: formatting
description: >-
  Personal Python code style (import grouping/order, NumPy-style docstrings,
  trailing-docstring module constants, lowercase comments, double-quoted
  strings). Use whenever writing or editing any Python code.
---

# Python code formatting

Apply these whenever you write or edit Python. They are intended to be
compatible with `black`, `isort` (profile "black"), and `flake8`
(max-line-length 88, ignoring E203/E501). If the repo has pre-commit, run
`pre-commit run --all-files` before committing.

## Imports

- All imports go at the **top** of the module — never inside functions (except
  to defer heavy/optional dependencies; follow existing patterns).
- Group in this order, separated by a blank line:
  1. standard library
  2. third-party (numpy, pandas, matplotlib, …)
  3. local (this package's modules)

```python
import numpy as np
from collections import Counter
import pandas as pd

from sklearn.cluster import KMeans
import matplotlib.pyplot as plt

from my_package.schema import DataLoader
from my_package.constants import AttrType
```

## Docstrings

NumPy-style, with `:` after section headers. Always include Parameters and
Returns.

```python
def my_function(param1: str, param2: int) -> bool:
    """Brief description.

    Parameters:
    -----------
    param1 : str
        Description of param1.
    param2 : int
        Description of param2.

    Returns:
    --------
    bool
        Description of return value.
    """
```

## Module-level constants

Document module-level constants with a **trailing docstring** (a string literal
on the line(s) immediately after the assignment), not a leading `#` comment.
Start it with the type, then a colon and the description. Use a trailing `\` to
wrap long lines.

```python
DICT_SCORE_KEY = {
    ScoreDatabase.Conservation: "score",
    ScoreDatabase.AlphaMissense: "amPathogenicity",
}
"""dict[ScoreDatabase, str]: Key holding the numeric score within each database's \
    score dict; databases absent from this mapping have no single scalar score."""
```

## Comments

Lowercase first letter for non-proper-noun comments:
`# create bins for the count column` (not "Create bins...").

## String quotes

Double quotes by default. Exceptions:
- nested strings inside an f-string use single quotes: `f"Value: {d['key']}"`
- docstrings use triple double quotes `"""`

```python
# preferred
filename = "my_file.csv"
color = "#FFD700"
result = f"The value is {my_dict['key']}"

# avoid
filename = 'my_file.csv'
```
