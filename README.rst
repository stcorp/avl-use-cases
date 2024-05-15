Creating a conda environment
----------------------------

Create new avl conda environment with::

  $ conda env create -f environment.yml

Then activate the environment with::

  $ conda activate avl-use-cases


After updating the environment.yml you can update the 'avl' environment using::

  $ conda env update -f environment.yml

To use a different environment name use e.g.::

  $ conda env create -n my-env-name -f environment.yml


Running jupyterlab
------------------

Start jupyterlab within your conda environment with::

  $ jupyter-lab

Within jupyter-lab select a use case .md file and select 'open as notebook'.
When you then save the opened notebook this will automatically generate a .ipynb file.
Any modifications in the .md or .ipynb file will be automatically synchronized to the other file.

You can also have jupyterlab open .md files as notebooks by default by using:

  $ jupytext-config set-default-viewer markdown
  $ jupytext-config list-default-viewer
  markdown: Jupytext Notebook


Creating HTML page
------------------
A fully rendered notebook can be turned into HTML using::

  $ jupyter-nbconvert --to html notebook.ipynb

A one liner that goes directly from .md to .html is::

  $ cat usecase01/Usecase_1_S5P_SO2_La_Soufriere.md \
  | jupytext --from md --to ipynb \
  | jupyter nbconvert --execute --stdin --to html --output usecase01/Usecase_1_S5P_SO2_La_Soufriere.html
