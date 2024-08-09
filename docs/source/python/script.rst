Using antelope in a script
==========================

We also provide a means of connection to your database via a script. This can be very helpful if you want to automatically parse a large amount of pre-existing data and insert it into your database, without having to go through the gui.

Connecting to the database
--------------------------

We do not want to expose database credentials in a script, or write them to disk at all. The safest way to access credentials in a script is via environment variables. It is therefore necessary that you set the `DB_USER` and `DB_PASS` environment variables in the execution environment before running the script. This could be done with `export` in bash, and these can be copied into a slurm environment by adding the following line to your slurm script for HPC use::

#SBATCH --export=DB_USER,DB_PASS

Within your script, you then want to establish a database connection. This can be done as follows::

   from antelope.load_connection import *

This reads all the tables and analysis functions for direct use in the script, just like in :ref:`interactive`.

Custom helper functions
-----------------------

Certain aspects of antelope's database integrity are enforced at the application level. Some parts of our schema design are not completely simple to interact with. For these reasons, we expose several helper functions, which are the recommended way to interact with certain parts of the database. By understanding all of these, it will be fully possible to use antelope in a fully programmatic way, without the gui.

However, it is possible to use the python connector to interact without these helper functions, which could lead to your database being in an inconsistent state. We therefore recommend at least understanding these functions if you are going to insert complex data programmatically.

Split trials
^^^^^^^^^^^^
We designed the masking functions to allow for the splitting of sessions into individual trials. This splitting is a common operation so we provide a function to efficiently do this for you. This function takes as arguments the event data and timestamps you want to split, and the mask data and timestamps, both of which should be a (data, timestamps) tuple. The data can be from any of AnalogEvents, DigitalEvents, IntervalEvents or Kinematics::

   from antelope import split_trials

   mask = (Mask & restriction).fetch1('data','timestamps')
   events = (AnalogEvents & restriction).fetch1('data','timestamps')

   trials = split_trials(events, mask)

This will return a list of (data, timestamps) tuples, each corresponding to a single trial, as defined by the mask.

Insert nwb
^^^^^^^^^^
One data insert that is potentially non-trivial is inserting nwb files. We batch insert all the different data for a single session within the behaviour schema, as they are all interelated and should come from the same nwb file. This is not a simple process. In particular, we took great care in desiging our json schema, that describes your behaviour rig, its geometry and features, and maps the data from our acquisition's nwb output into the correct tables. We therefore expose a function that reads this nwb file, the json file, and the database key, and performs all the necessary inserts.

This function is called `insert_nwb` and takes the following arguments:

