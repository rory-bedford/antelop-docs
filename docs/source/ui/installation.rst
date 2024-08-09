.. _installation:

Access/installation
-------------------

How you access the GUI depends on your infrastructure and how your database administrator setup Antelope. The recommended approach is to install Antelope locally via pip.

Local installation
^^^^^^^^^^^^^^^^^^

Antelope can be installed locally on your machine via pip. This is the recommended approach for most users.

Antelope requires python3.9. We **strongly** recommend using a virtual environment for your installation. Many users will use conda to manage virtual environments. To setup your environment and run antelope::

    conda create -n antelope python=3.9
    conda activate antelope
    pip install antelope[gui]

With antelope installed, you can now run the GUI by typing `antelope` in your terminal, in the activated environment. Just like a Jupyter notebook, this will run a local server and will open the application in your browser.

Optional dependencies
"""""""""""""""""""""
Additional dependencies can be installed to enable extra features in antelope, but these are not required for basic functionality. We provide them as additional dependencies, since these dependencies can be difficult ot install on certain systems.

In particular, if you want to use phy from within antelope, use the phy extra::

    pip install antelope[gui, phy]

And if you want to use deeplabcut's gui tools, run::

    pip install antelope[gui, deeplabcut]

To install all optional dependencies, run::

    pip install antelope[full]

Note the the lightweight version with no additional dependencies just allows you to use antelope's :ref:`python`.

Configuration
"""""""""""""

On first use, you'll need to run `antelope-config` to configure your app. This process includes linking antelope to your lab's database, your compute cluster, and local storage.

In order, the steps are:

- MySQL host: The host address of your lab's MySQL database.
- S3 host: The host address of your lab's S3 store.
- Local storage:

  - You can add many local storage locations which will be searchable within Antelope.
  - Each folder must first be given a name, such as 'Documents', which will be used to identify it in Antelope.
  - It must then be given an absolute path, in either Windows or Unix format.

- Analysis scripts:

  - Custom analysis scripts are written and kept in folders on your local machine.
  - You must specify the absolute paths to these folders here.

- Number of cores: Antelope runs many operations in background processes. You can specify the number of cores to use here. This must be an integer greater than 0.
- Cluster host: The hostname of your compute cluster, as you would use in an SSH command.
- Cluster antelope install path: The path to the antelope installation on your cluster. This will be given to you by your antelope administrator. Must be an absolute Unix path.
- Cluster data directory:

  - Most clusters have a way of mounting your cluster storage onto your local machine. Antelope needs to know the path to this directory on your local machine.
  - This should be an absolute Windows or Unix path.
  - If mounting is not possible, you won't be able to manually curate data in phy and track cluster jobs progress, but all other antelope features will still work.

If you want to manually edit your configuration file, it is stored in ~/.config/antelope/config.toml on Unix, and in %APPDATA%/antelope/config.toml on Windows.


Desktop app on shared machine
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Antelope can be run on a shared linux machine, such as a cluster head node. However, users need to access Antelope from their own machines. Since Antelope is a graphical application, we recommend using `X2Go <https://wiki.x2go.org/doku.php>`_, which enables you to run graphical applications on a remote machine. You should install the X2Go client on your Windows, Mac or Linux machine.

Inside the X2Go Client, you should start a new session, which you can name 'Antelope'. You then need to enter the host machine on which Antelope is installed, and your username. Under 'Session type', you want to select 'Single application' in the dropdown box. You then need to enter the command that runs Antelope, which should be of the form::

    /path/to/antelope.sh

Where your database administrator will give you the path for your particular installation.

Additionally, you should click on the media tab and uncheck 'Enable sound support'.

Optionally, under the shared folders tab, you can select data on your local machine that you want to use within Antelope. This could be a folder on your machine where you store trial data, for example. To do this, click on the folder icon and navigate to the folder you want to use in Antelope, then click 'Add'. Make sure you select the 'Automount' checkbox. This data will show up under the 'local' folder inside Antelope.

Once you're happy with all your options, click 'Ok'. We then recommend that you make a desktop shortcut to Antelope for easy access. Click on the options button for the Antelope session, and select 'Create session icon on desktop'.

With this initial setup complete, you now just need to click on the Antelope desktop shortcut, and enter your password, and Antelope will boot. There is no further setup required - everything should just work out of the box, including tools such as Phy, scheduling jobs on the cluster, and connecting to your lab's database. Note Antelope can take a while to boot up - sometimes around fifteen seconds.

Persistent webapp
^^^^^^^^^^^^^^^^^

If you are using a persistent web interface hosted on a dedicated lab server, you will simply need to navigate to the URL in your web browser given to you by the database administrator from within your institution's network (or using a proxy or VPN if outside the network).

Note however that the web version of antelope is slightly more limited - you cannot display certain graphical applications such as phy, and may not be able to schedule jobs depending on whether your cluster is directly accessible via SSH.
