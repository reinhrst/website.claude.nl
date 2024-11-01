---
title: How to structure a python project with multiple entry points
description: Guide on how to set up your python project so that you can call libraries from parent directories
tags:
  - tech
  - howto
header:
    image: header.webp
---

Already almost 10 years ago I asked [this question on Stack Overflow][1], and did not get a satisfactory answer.
Recently I had someone asking the exact same question in a Discord channel, and I decided to dive into this once more, and explain how I think it should be fixed anno 2024.

<!--begin-summary-->

In a python project, you might have multiple command-line-tools, each in their own subdirectory, that all need access to some shared code.
Python will not allow one to use relative imports to directories higher than the entry-point. So starting your entry-point with `import ..lib` will directly fail with
```
ImportError: attempted relative import with no known parent package`.
```
In this post I explore what I think is the most pythonic solution, and then some other solutions.

<!--end-summary-->

### Previous attempts

In order to explain the exact situation we're looking at, consider the following directory structure for a [spam][2]-app (or see the [SO question][1] for a more realistic example):

```text
/quickscripts/
  some_spam.py
/longscripts/
  continuous_spam.py
/lib/
  __init__.py
  libspam.py
```

This code has two entry-points (I would like to be able to run both `python quickscripts/some_spam.py` and `python longscripts/continuous_spam.py`), and both entry-points need access to the functions in `/lib`.

A naive approach would be to start `some_spam.py` with the line `from .. import lib`.
However this is not allowed and results in the previously mentioned `ImportError: attempted relative import with no known parent package`. (and no, adding a `__init__.py` to the top-level directory does nothing to avoid this error).


In the [Stack Overflow post][1] I mentioned a number of possible workarounds, and I remember that 10 years ago I opted for solution number 3: add three lines at the top of every entry-point adding the parent directory to `sys.path`:

```
import os
import sys
sys.path.append(os.path.join(os.path.abspath(
    os.path.dirname(__file__)), ".."))
```

Even though it worked quite well, I was never quite satisfied with the solution, _and it is not how I would do it next time_.

## Preferred solution

The 2024-Claude preferred solution is to make the code a proper package, by adding a parent directory and a `pyproject.toml` (which would also contain any dependencies the project may have; this example project actually used [`clargs`, my package for command-line apps][3]):

```text
/spam/
  quickscripts/
    some_spam.py
  longscripts/
    continuous_spam.py
  lib/
    __init__.py
    libspam.py
pyproject.toml
```

The `pyproject.toml` can contain the bare minimum:

```toml
[project]
name = "spam"
version = "0.1.0"
dependencies = [
    "clargs",
]
```

Right now you may install the `spam` package to the current `venv` in development mode by typing `pip install -e .` while being in the directory where `pyproject.toml` resides.
Having the package installed in development mode means that any changes we make in the files (e.g. in the `libspam.py`) will instantly be available, without having to do `pip install .` again.

After the package is installed, the files `some_spam.py` and `continuous_spam.py` could get to the library using `from spam import lib`.
Even though this solves our problem, I don't quite like this solution; `from .. import lib` shows much better where the import comes from (and also cannot be misunderstood if there is some other package `spam` installed).

Right now `from .. import lib` works, however _not when the file is called directly from the command-line_.
But if we call the file as a module, then it works: `python -m spam.quickscripts.some_spam -s 10`.

You can see an [example of the full code on GitHub][4].

Obviously one could make an alias to `python -m spam.quickscripts.some_spam` to limit the typing work.

## Bonus points

I think there is an even slightly better way to do this: [`pyproject.toml` has a way to define entry-points][5].

A slight change to the code is necessary, since this `[project.scripts]` section can only call methods, not modules.
A quick way to fix this is just replace the `if __name__ == "__main__":` in your entry-point files with `def main():`.

```python
from .. import lib


if __name__ == "__main__":
    print(lib.get_spam())
```

becomes

```python
from .. import lib


def main():
    print(lib.get_spam())
```

Now in `pyproject.toml` we define the entry-points:

```toml
...

[project.scripts]
simple_spam = "spam.quickscripts.simple_spam:main"
some_spam = "spam.quickscripts.some_spam:main"
continuous_spam = "spam.longscripts.continuous_spam:main"
```

A small extra issue here is that every time `pyproject.toml` is edited, the package has to be installed again with `pip install -e .`.
To be clear: this is only necessary if you add or remove an entry-point; you can still edit the code as before, and changes are instantaneous.

Now the programs can be run by simply typing `simple_spam`, `some_spam` and `continuous_spam`.

The advantage of this approach over making aliases, is that anyone who now installs the `spam` package, will have these short commands.
Also its possible to have multiple entry-points per file, and it abstracts the exact location of the modules within the package.

I also [made an example available of this approach][6].

One could even make a version that works both with the `python -m ...` approach and this new approach:

```python
from .. import lib


def main():
    print(lib.get_spam())


if __name__ == "__main__":
    main()
```



[1]: https://stackoverflow.com/questions/28580962/structure-python-project-with-multiple-entry-points-and-centralised-config-fil
[2]: https://en.wikipedia.org/wiki/Spam_(Monty_Python_sketch)
[3]: https://pypi.org/project/clargs/
[4]: https://github.com/reinhrst/spam/releases/tag/call-as-module
[5]: https://packaging.python.org/en/latest/guides/writing-pyproject-toml/#creating-executable-scripts
[6]: https://github.com/reinhrst/spam/releases/tag/entrypoints-on-package
