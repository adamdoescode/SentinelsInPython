# Sentinel Values

As of python 3.15 the standard library [implements a new class called sentinel](https://docs.python.org/3.15/library/functions.html#sentinel).

This repo holds a README which serves as my notes in preparation for a very short talk about sentinel values in Python for the PythonWA July 2026 meetup. It's currently otherwise vacant but may end up with a modest python3.15 uv env and a script... depends how much time I have tonight.

## A very brief introduction

## What are sentinel values?

They are unique placeholder values. As [pep 661 states](https://peps.python.org/pep-0661/), they have many uses:
- Default value for function arguments when no value is given:
```python
def foo(value=None): ...
```
- Return values from functions when something is not found or unavailable:
```python
"abc".find("d")
-1
```

## Semantics

Taken from the [pypi `sentinels` package](https://pypi.org/project/sentinels/). I believe these attributes line up with those of the python3.15 implementation.

Sentinels are always equal to themselves:

```python
>>> NOTHING == NOTHING
True
```

But never to another object:

```python
>>> from sentinels import Sentinel
>>> NOTHING == 2
False
>>> NOTHING == "NOTHING"
False
```

Copying sentinels returns the same object:

```python
>>> import copy
>>> copy.deepcopy(NOTHING) is NOTHING
True
```

And of course also pickling/unpickling:

Adam note: why note the pickling here? Turns out a singleton sentinel unpickled will be a new object which is not a singleton.
You have to implement `__reduce__` to make it return the OG object.
Also it wont return true for `NOTHING == NOTHING'.

```python
>>> import pickle
>>> NOTHING is pickle.loads(pickle.dumps(NOTHING))
True
```

## Examples

- In python, `None` is the classic sentinel value.
- Standard library python has other sentinels - as we can see from above that str.find() returns `-1` as a sentinel.
- `np.nan` from [the numpy constants](https://numpy.org/doc/stable/reference/constants.html) and, arguably, `np.inf`.
- [This post](https://mail.python.org/archives/list/python-dev@python.org/message/JBYXQH3NV3YBF7P2HLHB5CD6V3GVTY55/) compiles a dozen examples from the python standard library.

First item on the list is an example of using a sentinel in the stdlib.
In `MutableMapping`, aka part of the code for Dictionary object:

```python
    __marker = object()

    def pop(self, key, default=__marker):
        '''D.pop(k[,d]) -> v, remove specified key and return the corresponding
        value.  If key is not found, d is returned if given, otherwise
        KeyError is raised.
        '''
        try:
            value = self[key]
        except KeyError:
            if default is self.__marker:
                raise
            return default
        else:
            del self[key]
            return value
```

In practice, the above implements `dictthing.pop(key, default)` where if
you dont pass a default, and thus `default is __marker` it
raises `KeyError`.

## What does PEP 661 do?

PEP 661 introduces a specification for 

## An example relevant to me

A common environmental science question:
>Given a small pile of dirt how much of it is lead?

- Analytic tests have a certain level of sensitivity - that is, they can only be so accurate.
- If the concentration measured is below the level where the test is considered accurate its no longer possible to know if the measurement is zero or not because the range of possible values now includes zero.

Here's a sample dataset of soil lead concentration measurements, with two results flagged as below the EPA Method 7000B (Flame Atomic Absorption Spectroscopy) detection limit of approximately 0.5 mg/kg:

| Date | Lead Concentration (mg/kg) | Result Flag (limit = 0.5mg.kg) |
|------------|----------------------------|------------------------|
| 2023-01-01 | 12.4                       | Detected                |
| 2023-04-01 | 0.3                         | Below Detection Limit   |
| 2023-07-01 | 18.7                       | Detected                |
| 2023-09-01 | 0.2                         | Below Detection Limit   |
| 2023-12-01 | 9.6                         | Detected                |

So, what do we **do** with these "below detection" values?
- We could treat them as the supplied value...
  - but what if they are actually zero?
- We could treat them as zero...
  - but then the average will be wrong:

```python
>>> results = [12.4, 0.3, 18.7, 0.2, 9.6]
>>> results_zeroed = [12.4, 0, 18.7, 0, 9.6]
>>> np.mean(results)
8.24
>>> np.mean(results_zeroed)
8.14
```

- We could treat them as nan...
  - this is better since we are no longer biasing our average
  - but this still throws away the actual value measured for our below detection measurements.
  - but `np.nan` propogates when we take an average:

```python
>>> results_naned = [12.4, np.nan, 18.7, np.nan, 9.6]
>>> np.mean(results_naned)
np.float64(nan)
```

In short, we are stuck with a set of compromises!

We could consider implementing our own sentinel value using the python 3.15 sentinels.
