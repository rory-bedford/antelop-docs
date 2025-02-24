Access/installation
===================

Local installation
^^^^^^^^^^^^^^^^^^
The local installation instructions are identical as for the web app, listed in :ref:`installation`. The difference is that it requires fewer dependencies, so can be installed more quickly. If you have already installed the web app, this should be working already.

The configuration is also identical as for the web app. However, the only configs necessary are the MySQL and S3 hosts, and the analysis scripts directory. If you're not going to use the gui, the rest can all be left blank.

Cluster installation
^^^^^^^^^^^^^^^^^^^^
The cluster installation is provided as a singularity container. This should have been set up and configured by your database administrator. The container can be run as an executable if you have singularity installed, so you just need to execute::

    /path/to/antelop-python.sif

If you're going to use this method a lot, we recommend using a bash alias to make this easier. In your bashrc, just add::

    alias antelop-python='/path/to/antelop-python.sif'
