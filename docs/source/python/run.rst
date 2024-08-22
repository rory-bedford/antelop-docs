Analysis framework
==================

Antelope provides a custom framework for writing and running analysis functions on your database. Features this framework provides include automated running on different database keys, tight integration with the gui, and tools to aid reproducibility of a function run. We also provide a comprehensive standard library of common analyses.

Running analysis functions
--------------------------

Analysis functions are very straightforward to run. They take in as their first argument a key that restricts the function to run on a subset of your database. In the absence of any key, the function will run on the entirety of the database. The key can be a dictionary or a string, in datajoint syntax. They then take as optional keyword arguments the additional arguments to the function. Functions always return a list of dictionaries, with the primary keys it ran on and the results. These can easily be loaded into a pandas dataframe using `pd.DataFrame(result)`.

As an example, consider running the first function in our demonstration script in the analysis standard library::

    hello_world.greeting()

For our lab's database, this returns:

+-------------+------------------------------+
| experimenter| greeting                     |
+=============+==============================+
| arueda      | Hello, Ana Gonzalez-Rueda!   |
+-------------+------------------------------+
| dmalmazet   | Hello, Daniel de Malmazet!   |
+-------------+------------------------------+
| dwelch      | Hello, Daniel Welch!         |
+-------------+------------------------------+
| ewilliams   | Hello, Elena Williams!       |
+-------------+------------------------------+
| fmorgese    | Hello, Fabio Morgese!        |
+-------------+------------------------------+
| mtripodi    | Hello, Marco Tripodi!        |
+-------------+------------------------------+
| rbedford    | Hello, Rory Bedford!         |
+-------------+------------------------------+
| srogers     | Hello, Stefan Rogers-Coltman!|
+-------------+------------------------------+
| yyu         | Hello, Yujiao Yu!            |
+-------------+------------------------------+

Or alternatively, changing an argument and running on a restriction::

    hello_world.greeting({'experimenter':'rbedford'}, excited=False)

Gives:

+-------------+------------------------------+
| experimenter| greeting                     |
+=============+==============================+
| rbedford    | Hello, Rory Bedford.         |
+-------------+------------------------------+

.. _reproducibility:

Reproducibility
---------------

The simple method of running a function described above works well for small analyse, and for testing while developing a function. However, a typical use case we envisage is that you would test a function on, say, a single session like so, but then run your analysis on all sessions belonging to a single experiment, potentially involving months' worth of recordings. In this case, you would presumably run your function on a cluster, and save your results to disk, for further analysis, visualisation, and publication. To do so, it is important to record *exactly* what function was run, on what data, with what parameters, etc.

For these reasons, we provide a toolkit aimed at making exact the exact reproduction of a function run easy. We see this as a core feature of Antelope. Reproducibility within neuroscience can be very difficult in general, and the advent of the big-data era can make this problem even worse. However, with proper data engineering practises, this problem can in fact be solved, and our intention with this framework is to make this not only possible, but easy for all users to make a routine part of how they publish data.

In particular, the important things to consider in order to make an analysis routine exactly reproducible are:

* Data consistency: the data you are analysing should be fixed in time, and not change between runs.
* Function consistency: the code of the analysis function should also be exactly fixed.
* Parameter consistency: the parameters passed to the function should be recorded exactly.
* Dependency consistency: the exact environment in which you are running your analysis should be recorded.

Now, some of these challenges are already addressed by widely used tools in the community. For example, for the final point of dependency consistency, tools such as virtual environments (eg conda), or containers (eg singularity), should be used to exactly reproduce the environment in which the function was run, and are widely distributed with the code repository accompanying a publication. Data availability is addressed by a number of tools as described in `DataJoint's documentation <https://datajoint.com/docs/core/datajoint-python/0.14/publish-data/>`_. For example, you can provide access to your MySQL database to the community, or export your data to a format such as NWB for sharing, or containerise your database in Docker with a SQL dump so users can run their own instance of your database. We provide additional functionality on top of these methods, designed to be used alongside them, that rigourously checks the consistency of the data when sharing for a publication.

Reproducibility file
^^^^^^^^^^^^^^^^^^^^

We provide a method to run your analysis function that saves the results to disk, alongside an additional metadata file, that enables exact reproducibility of the function call, as per our criteria discussed above.

This file is json-based, and includes the following information:

* `name`: the name of the function that was run
* `restriction`: the restriction that the fucntion was run on
* `arguments`: the arguments passed to the function
* `data_hash`: a hash of all data in the database the function could make use of
* `code_hash`: a hash of the code of the function to check it hasn't changed

Of particular importance is the data hash. Recall that each function in antelope specifies the database tables it will run on, the key it will use to fetch data, and other functions it can call. Our algorithm uses the MD5 hashing algorithm on all rows in the tables matching the function call's key that are available to the function. The hash runs on all table attributes, including blobs. The function can't run on any data outside these rows, so checking their hash is sufficient to ensure the data hasn't changed. One complication is that other functions called by the parent function can depend on different tables to the parent; our algorithm therefore traverses the graph of function calls to collect all tables the parent function depends on, including those from arbitrarily deep other function calls.

A quick note on hashing results: we do not want to enforce that your analysis routines are necessarily deterministic, so we do not check a hash of your results. If you want to add this and your function is definitely deterministic, it would be very straightforaward to compute a hash of the pickle file of the results.

Saving data with reproducibility
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each antelope analysis function can be called via an additional method, called `save_result`, that writes both the results and our custom file to disk. This takes the same arguments as the function call itself, with an additional argument called 'filepath' that specifies where to save your results to. Results are always pickled to save them to disk, and should therefore end with `.pkl`, potentially with additional extensions specifying the compression to use if desired. These can then be loaded into a pandas dataframe as follows::

    import pandas as pd
    result = pd.read_pickle('results.pkl')

By default, filepath is set to `./result.pkl`. In addition to the result, the reproducibility file is saved with the same name and location but the json extension. This should then be distributed along with the results.

For example, to run the first function in our demonstration script, saving in the current directory, on the entire database, with default parameters, you would run::

    hello_world.greeting.save_result()

To specify parameters::

    hello_world.greeting.save_result('./result.pkl', {'experimenter':'rbedford'}, excited=False)

Validating a function run
^^^^^^^^^^^^^^^^^^^^^^^^^

To validate the integrity of all the above factors before rerunning an analysis function, we provide the `check_hash` method. This takes as input just the reproducibilty json file. It checks the data_hash and code_hash against the database and function definition, and returns a message describing what's changed, or whether you car rerun the function. For example::

    hello_world.greeting.check_hash('./result.json')

Returns::

    Reproducibilty checks passed

Rerunning a function
^^^^^^^^^^^^^^^^^^^^

A similar method to the one above reruns the analysis function and saves the results, after checking the hashes as above. This method takes in both the saved json, and an output to save results to. Execute as follows::

    hello_world.greeting.reproduce('./result.json', './result.pkl')

This allows you to reproduce the results of an analysis.
