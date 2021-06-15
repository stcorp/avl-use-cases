Creating a conda environment
----------------------------

Create new 'avl' environment with::

  $ conda env create -f environment.yml

Then activate the environment with::

  $ conda activate avl


After updating the environment.yml you can update the 'avl' environment using::

  $ conda env update -f environment.yml

To use a different environment name use e.g.::

  $ conda env create -n avltest -f environment.yml


Running jupyterlab
------------------

Start jupyterlab within your conda environment with::

  $ jupiter-lab

Within jupiter-lab select a use case .md file and select 'open as notebook'.
When you then save the opened notebook this will automatically generate a .ipynb file.
Any modifications in the .md or .ipynb file will be automatically synchronized to the other file.


Creating HTML page
------------------
A fully rendered notebook can be turned into HTML using::

  $ jupyter-nbconvert --to html notebook.ipynb

A one liner that goes directly from .md to .html is::

  $ cat usecase1/notebook.md \
  | jupytext --from md --to ipynb \
  | jupyter nbconvert --execute --stdin --to html --output usecase1/notebook.html
