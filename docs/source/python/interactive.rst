.. _interactive:

Interactive use
===============

Basic usage
-----------
To use antelope's interactive mode, run the command `antelope-python` in the terminal, either in your conda environment, or with your singularity container alias set.

This command will prompt you for your database username and password. It will then establish a connection to the database, and will open an interactive IPython shell. All the database tables are automatically available as global variables. All the analysis functions, from both the standard library and your own analysis folders are available with the syntax `script_name.function()`.

DataJoint Queries
-----------------

You can interact with the database using DataJoint syntax. You can query the database using the following operators:

.. csv-table:: DataJoint Query Operators (source: DataJoint documentation)
   :header: "operator", "notation", "meaning"

   "restriction", "A & cond", "The subset of entities from table **A** that meet condition **cond**"
   "restriction", "A - cond", "The subset of entities from table **A** that do not meet condition **cond**"
   "join", "A * B", "Combines all matching information from **A** and **B**"
   "proj", "A.proj(...)", "Selects and renames attributes from **A** or computes new attributes"
   "aggr", "A.aggr(B, ...)", "Same as projection but allows computations based on matching information in **B**"
   "union", "A + B", "All unique entities from both **A** and **B**"

To perform a query, you just type your query and press enter.

For example, consider the following query::

   Experiment * Experimenter & 'experimenter="rbedford"'

Which gives the result:

.. csv-table::
   :header: "experimenter", "experiment_id", "full_name", "group", "institution", "admin", "experiment_name", "experiment_notes", "experiment_deleted"

   "rbedford", "1", "Rory Bedford", "Tripodi Lab", "MRC LMB", "True", "experiment 1", "Example notes, experiment was a terrible failure.", "False"
   "rbedford", "2", "Rory Bedford", "Tripodi Lab", "MRC LMB", "True", "experiment 2", "Experiment was a great success, I won a Nobel prize for this.", "False"

Schemas
-------

We list show here the tables available in the database within their schema structures. To see their attributes, you can run the `.describe()` method.

.. tabs::
   .. tab:: Metadata schema
      .. image:: ../images/session.png
         :alt: Antelope metadata schema

   .. tab:: Electrophysiology schema
      .. image:: ../images/ephys.png
         :alt: Antelope electrophysiology schema

   .. tab:: Behaviour schema
      .. image:: ../images/behaviour.png
         :alt: Antelope behaviour schema

Analysis functions
------------------

Also available in interactive mode are the analysis functions. These are accessed with their script name and function name separated by a dot. For example::

   hello_world.greeting()

Will run the function `greeting()` from the script `hello_world`, on all the data in your database. This will return a list of dictionaries, each containing the primary keys and results of the analysis function. This can be easily converted to a pandas DataFrame for further analysis as follows::

   import pandas as pd
   df = pd.DataFrame(hello_world.greeting())

Analysis functions accept as their first argument a DataJoint restriction. This allows you to run the analysis on a subset of the data. For example::

   hello_world.greeting({'experimenter':'rbedford'})

Additionally, they can accept any number of keyword arguments, that modify the behaviour of the function. For example::

   hello_world.greeting({'experimenter':'rbedford'}, excited=True)

To see what keyword arguments are available, check the `args` attribute as follows::

   hello_world.greeting.args

To see all functions available in antelope's standard library, see :ref:`stdlib`.

Note, if you are writing analysis functions, it is useful to be able to reload them on the fly, without having to close and reopen your antelope shell. To do this, just run::

    reload()
