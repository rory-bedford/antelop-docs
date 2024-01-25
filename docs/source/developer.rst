Development guide
=================

This section of the documentation details the general development goals of Antelope. In particular, we discuss the structure of the repository, our security model, the structure of the codebases, and the structure of the Nextflow pipelines. This is so that future collaborators can be quickly brought up to speed on the project.

Repository structure
--------------------

Antelope involves a few distinct components that undergo separate development, need separate installations on the various host machines, but are all interconnected when Antelope is deployed. To address this, we manage different **orphaned** branches in the repository, which are branches that share no history. This allows us to keep the different components of Antelope in a single github repository. Despite this not being the most common use case of branches, we feel it's simpler than the alternative solutions of managing distinct repositories for the different components, or of having all the different components in subdirectories within the main branch.

The branches are as follows:

* Main - this branch should be seen by users wanting to install Antelope. It contains installation scripts, containers, and a README to help users get started with Antelope as easily as possible. It is designed so that users don't need to access any other branch.
* Docs - this branch contains all the documentation, built automatically and hosted at https://antelope.readthedocs.io/en/latest/setup.html.
* Gui - this branch contains all the code for the graphical user interface
* Python-package - this branch contains the code for the python package
* Workflows - this branch contains the Nextflow scripts which need to be installed on the computational server/HPC.

Other branches may be created for development purposes, as we implement a new feature, before merging with their parent branch. These branches will be clearly named with the parent branch they belong to and the feature they are implementing.

Security practices
------------------

Antelope is designed to be used in academic institutions. Our security model is therefore based on existing tools designed for this purpose - such as, for example, Nextflow secrets management. Academic institutions can be difficult in that a number of different users have access to the same machines. Our priority is that any non-root users on the same machine as any component of Antelope cannot intercept any database credentials.

In particular, we use the following database accounts for the MySQL database and S3 object store:

* The root user, whose credentials should obviously be kept securely by the database administrator.
* A single user with SELECT, INSERT, DELETE and UPDATE priviledges on all Antelope tables, used under the hood by all lab members. These define the credentials used by the webapp and nextflow pipelines to connect to the database. At the application level within the web interface, and within the Nextflow pipelines, we implement what amounts to row-based priviledges so users can only INSERT, DELETE and UPDATE their own data, in a manner that maintains the database integrity.
* Users with just SELECT priviledges for using the antelope-python package.
* Optional additional users with just SELECT priviledges for sharing data with collaborators.

The user interface itself requires a separate per-user password. This is how we enforce permissions at the application level. We use streamlit-authenticator for this purpose, which stores a hash of all user passwords inside the interface container, and checks the password against this hash whenever the app is run. It is up to the users to store this password safely to protect their own data.

The database credentials are also stored inside the application container. They are copied into the container along with other configuration files at build time. It is therefore also necessary to manage access to the antelope container. For a web app deployment, the docker container will typically be hosted on a private lab server, so this shouldn't pose a problem. For the desktop app deployment, it is important to manage permissions of the singularity container, as it will typically be kept in a publicly accessible location. We use a unix group for this purpose: the container belongs to our lab unix group on the HPC, and has read/execute permissions for this group, and no permissions for other users. It is possible for a lab member to manually shell into the container and read the credentials, then modify other members' data using a different MySQL connector. We therefore ask lab members not to do this, and trust them to only interact with the database using either the GUI or the python-package.

The python package is based on DataJoint in terms of its syntax. However, we typically do not want users modifying the database through the python package, as our pipelines are defined to be triggered through the web interface, which also implements row-based permissions, and takes additional steps to enforce database integrity. For that reason, we have removed DataJoint's insert(), update() and delete() functions in our python package, and additionally supply users with the SELECT only credentials for using this package.

The Nextflow pipelines will almost certainly be run on a shared machine. The database credentials are injected into the pipeline environment as environment variables when triggered from the GUI, then called inside nextflow as environment variables to be injected into the work environment for eahc process on the HPC, so they can be accessed by the scripts that need to interact with the database. So again, this is another means by which lab members could access the database credentials (by shelling into the nextflow process and listing all environment variables), and we need to trust that they will not abuse this.

For the python package, users have two options. When first running the package, users will be prompted for their login details, which will be saved as temporary variables in the dj.config dictionary. If on a shared machine, it is recommended that users login like this every time they use the package. If running on their own machine, users can save their login details to disk using either dj.config.save_local() or dj.config.save_global(), in which case they won't need to login each time they use the package.

The final security point worth mentioning is that when a user submits a computational job to run, the GUI needs to issue a command to the cluster entry node, or other compute server, telling it to run the predefined Nextflow pipeline. To do this, we use the paramiko python package to issue the command via ssh. This is also necessary in the desktop app deployment to shell out of the container to execute the pipeline on the host. The cluster entry node or compute server address is pre-configured by the database administrator in the configuration file, or in the desktop app deployment, it just SSHs to the host the container is run on. The user needs to manually enter their password (and optionally username if it's different to their Antelope username), through streamlit's text_input widget in password mode. We decided it is not secure to store these credentials anywhere at all, and it acts as a good check for the user that they really want to submit that job anyway.

Graphical user interface code structure
---------------------------------------

The graphical user interface code is held inside the Gui branch of the repository. It contains all the code needed to run the interface as either a desktop app or a web app. Additionally, this branch contains the **Dockerfile** for the web app deployment, the singularity definition file **antelope.def** for the desktop app deployment, and the **requirements.txt** or **environment.yml** for setting up a development environment with pip or conda.

The GUI is written as a streamlit app, which is a user-friendly python web development framework. The 'desktop app' deployment option essentially just bundles the web app with a minimal version of the chromium browser, and runs chromium in 'app' mode, which gives it a desktop-like feel.

At the root of this branch is the script **app.py**. This is the main script for the application, run by calling::

    streamlit run app.py

This script asks the user for login details, and if they're valid, opens a connection to the database, which stays open for the remainder of the web app session. Additionally, it imports all the pages to display, and performs some simple configuration of the web interface.

It's worth noting briefly at this point streamlit's data flow, which is unusual for a web development framework. Any time the user interacts with anything on the page, the entire script is rerun from top to buttom. Internal caching mechanisms allow streamlit to save the state of many widgets, and its very important to cache many objects such as database connections, so they're not recomputed each time the user interacts with the page. It can take a little while to get used to this (for example, nested buttons just don't work as you'd hope). This is a big change from typical web frameworks, where the backend usually involves an API which waits for requests from the frontend.

It's also worth noting that due to this model, the application is frozen to the user while long running computations occur. This is again why caching is so important, as it makes the page much more snappy. Additionally, we want to run some long running computations in the application, such as uploading a large trial recording to the database, which can take around 5 minutes. If this ran in the main streamlit process the user would have to wait for it to finish before they can do anything else. We therefore have a separate process pool to which these jobs are submitted, with a configurable maximum number of workers. In particular, we use this process pool for uploading, downloading, and modifying (restores and temporary deletes) data in the database. Often this is unnecessary - uploading a small single row is very quick, but we generalise this specifically for the cases where it's not quick.

In order for the app to run, it needs additional directories and files in the same directory as **app.py**: **resources/**, which contains a small amount of data used inside the app; **antelope/**, which contains the scripts used within the app; and a hidden directory **.streamlit/**, which contains configuration files. Additionally, **find_port.py** is used by the singularity runscript to find a free port on the host machine to expose the web app on, but is not needed to run the app locally during development or as a web app as the default port should be fine.

The .streamlit/ directory contains four files: **config.toml**, **secrets.toml**, **credentials.yaml**, and **antelope.toml**. In the main repository these contain placeholders, but they will need to be edited when installing antelope, to then get copied into the container. **config.toml** contains streamlit configurations. This can be left alone, or used to configure the theme of the app. **secrets.toml** contains the MySQL and S3 credentials. **credentials.yaml** contains the hashes of the individual user passwords. `Streamlit-authenticator <https://blog.streamlit.io/streamlit-authenticator-part-1-adding-an-authentication-component-to-your-app/>`_ details the structure of this file and how to generate new password hashes. **antelope.toml** contains antelope specific configurations, including how many threads the app should use, where to locate the nextflow pipelines, who has administrator permissions, etc.

Within the antelope directory, we find three subdirectories: **database**, **gui**, and **utils**. The antelope directory is itself a package, and the subdirectories are all subpackages that can be imported through the syntax::

    import antelope.gui.search

The database directory contains the scripts **dbstructure.py** and **dbconnect.py**. **dbconnect.py** has a few different functions, but it mainly just reads in the database credentials to establish a database connection, and returns the connection object. Additionally, it provides a cached version of this connection function, so that streamlit doesn't reestablish a new connection with each user interaction, and a validation mechanism to check the connection whenever retrieving it from the cache. Finally, it provides yet another wrapper for the connection function to be used in the separate threads that are triggered by the app. This simply checks if a connection has been established in that thread and is functioning, and either returns the thread connection or establishes a new one.

The gui directory contains a script for each page in the web gui. These scripts are not simple, as the pages have quite a lot of functionality, and we try to account for a large number of edge cases in terms of user interactions. They do, however, detail mainly just the flow of the page, calling functions from other scripts for any complex custom widgets or longer running functions that are sent off to different processes.

The utils directory contains utility functions that are reused across the web interface. We split this into four scripts: namely, **datajoint_utils.py**, **streamlit_utils.py**, **multithreading_utils.py**, and **external_utils.py**. **datajoint_utils.py** contains functions that add to DataJoint's functionality, including functions that parse the schema structure to return the parent or earliest ancestor of a given table attribute, and functions that perform queries in a manner suitable for displaying results in the web interface (for example, not downloading external store data to the working directory). **streamlit_utils.py** contains custom streamlit widgets that we reuse across pages. These often take DataJoint tables as input, so are not general streamlit widgets but are useful for the Antelope project. They include interactive interfaces to select different table entries, and interactive interface to select spike sorter parameters. **multithreading_utils.py** uses the library concurrent.futures to establish a single process pool in which to run long-running functions, such as inserting a large trial recording. It's important that the reference to this process pool is maintained as the user interacts with the app, so it is kept in the streamlit session state, along with different session states which hold the future objects necessary for checking the status of jobs in the process pool. Finally, **external_utils.py** is used for triggering anything that runs outside the main container, such as a nextflow pipeline, or a separate GUI app like phy.

Python package code structure
-----------------------------


Pipeline details
----------------

The pipelines are all in the Workflows branch of the repository.

Some general pipeline notes
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Antelope is built on top of DataJoint. However, it is not actually a DataJoint pipeline as DataJoint pipelines are intended to be constructed. The reason for this is that we found DataJoint to be lacking in functionality when it comes to how computations are performed. In fact, DataJoint computations, defined within a table's make() function, and called by running populate() on that table, are actually only designed to be run locally and sequentially. For large computations, such as running a number of spike sorters in parallel, this is very restrictive.

We therefore designed our computations as a set of distinct Nextflow pipelines. Each pipeline can be called from the web interface, and take the primary keys of the dependent entries as input parameters. Nextflow is a fantastic tool for complex pipeline construction as it allows dependencies between different jobs to be easily specified, and supports a range of computation engines to be used under the hood, which allows us to configure Antelope to run on a HPC using the SLURM job scheduler, or to run computations on a dedicated compute server, or to run them locally, on AWS, etc.

Our pipelines are all designed so that they first query the database to figure out which attributes to run for. For example, if you are spikesorting, you can perform a single computation by inputting a trial key, just like running a single datajoint make() computation. Alternatively, you can input an experiment or animal key, and then the computation will occur for all trials that belong to this key in parallel, up to the limits on cluster resources you have configured. This makes computation much, much quicker than running a datajoint populate() call, which will run computations one after the other. So the first nextflow process is this initial query, which fetches the keys deriving from the input parameter which have not been computed yet (such as all new trials belonging to the input experiment). These keys are then fed as different entries into a nextflow channel, and run in parallel for the rest of the pipeline. Future nextflow processes then download any raw data needed, perform computations, and upload the results, all again in parallel.

It is worth noting the challenges involved with running jobs on a HPC that pull data from a MySQL database. Namely, we do not know when a scheduled job will run. If the user schedules a job, then edits any of the dependent tables before the job runs, the job will fetch the data **as it is when the job runs**, not as it is when the job is scheduled. There is no simple way around this issue, so it is important that users are aware of this, or else they may get unexpected results. This in itself is not a major issue.

A bigger issue is the potential risk of the database being changed **during a computation**. This would lead to a loss of integrity of the database - the downstream populated tables could end up not actually being the results of computations on the data they depend on but on that data's historical value before the data was edited. This does pose a genuine risk when computations are long or cluster queues are long. The solution we use for this is to fetch all the necessary data for the computation at the start of the pipeline, and then lock the rows from which the data was fetched. We then perform our computations, upload the results at the end, and unlock the corresponding rows.

This also gets around the related issue of users being able to perform computations on overlapping queries. For example, if a user schedules a spikesorting on an animal, then shortly after they schedule spikesorting on the experiment that animal belongs to, the animal could end up having all its trials spikesorted twice, wasting resources, since the second job will see that that animal doesn't have any results yet, since its computation is still underway, and will also spikesort it. Therefore our initial database query checks the keys which haven't yet been computed over, and which are not currently locked. This allows for lazy user usage of the pipeline - you could just schedule spikesorting for your experiment each time you upload a new trial, and it will only run over the new data, avoiding any redundant calculations.

Another issue is that if many different entries are computed in parallel, they may all get inserted into the table at the same time, which could overload the database I/O and lead to performance issues for other users. It's therefore crucial when inserting results at the end of the pipeline that inserts are batched as far as memory allows. With varying runtimes for each computation, we haven't encountered issues with this yet, but if it does become a problem in the future, a possible solution would be to query the network traffic prior to an insert, and wait until it drops below a certain threshold before inserting.

It may seem slightly strange that we are relying on DataJoint at all at this point, given that we have essentially rewritten how computations are performed completely. There are a few reasons for this. DataJoint still has some features that we liked: namely, really nice integration with S3 'external' stores, and a nice python syntax that is familiar to many neuroscientists (which makes our python package very user friendly and extendible). We also like the general idea of having manual vs computed table types. Although this isn't necessary for Antelope, it acts as a nice clarification for users of how different tables in the pipeline function.

Pipeline specifics
^^^^^^^^^^^^^^^^^^

