Database Installation
=====================

Here, we will cover how to get up and running with the MySQL database and (optionally) the S3 store.

Hardware Requirements
---------------------

For optimal performance of the Antelop MySQL database, consider the following hardware recommendations:

- **CPU**: 4+ cores recommended. Database performance benefits from multiple cores for concurrent operations.
- **RAM**: Minimum 8GB, 16-32GB recommended for production environments. MySQL's performance is heavily dependent on having sufficient memory for caching and query processing.
- **Storage**:
  - SSD storage strongly recommended for database files
  - Minimum ~100GB for most lab MySQL databases
  - If using S3 storage, this will be substantially larger, and depends heavily on your requirements. Labs with high-throughput electrophysiology and imaging data may require around 100GB per day.
  - RAID configuration for error robustness (we use ZFS for this)
- **Network**: 1 Gbps or faster network interface for database servers, especially when hosting remotely
- **Backup Storage**: Additional storage equivalent to at least 2x your database size for backups

Software Prerequisites
----------------------

To run the Antelope database, you'll need:

- **Docker**: Version 20.10.0 or newer

  - `Docker Installation Guide <https://docs.docker.com/engine/install/>`_
  - Ensure the Docker daemon is running and you have permissions to use it

- **Docker Compose**: Version 2.0.0 or newer

  - `Docker Compose Installation Guide <https://docs.docker.com/compose/install/>`_
  - Note: Docker Desktop for Mac and Windows includes Docker Compose
  
- **Linux Distribution**: Any modern Linux distribution (Ubuntu 20.04+, Debian 11+, CentOS 8+, etc.)

  - While Docker runs on macOS and Windows, we recommend Linux for production database servers
  - Kernel version 5.4 or newer recommended for optimal Docker performance

- **Network Connectivity**: The server should have stable network connectivity with fixed IP or hostname

All other dependencies (MySQL, S3 components) are containerized and will be automatically pulled by Docker.

Installation
------------

We provide a very straightforward `docker-compose` setup for the database. You can find the `docker-compose.yml` file under `install_scripts/database/` in the Antelope repository. To pull this automatically to your server, run:

.. code-block:: bash

    curl -O https://raw.githubusercontent.com/marcotripodi/Antelope/main/install_scripts/database/docker-compose.yml

Or alternatively, if you want the no-S3 version, run:

.. code-block:: bash

    curl -O https://raw.githubusercontent.com/marcotripodi/Antelope/main/install_scripts/database/docker-compose-no-s3.yml

Now open this file in a text editor such as `vim` or `nano` and fill in root credentials. These credentials are, obviously, very important, so use a strong password, and store them somewhere safe.

With this done you can now boot up the database with:

.. code-block:: bash

    docker-compose -f docker-compose.yml up -d

Or:

.. code-block:: bash

    docker-compose -f docker-compose-no-s3.yml up -d

This will start the database in the background. You can check that it is running with:

.. code-block:: bash

    docker ps

You should see a container with the name `mysql` running.

To stop the database, run:

.. code-block:: bash

    docker-compose -f docker-compose.yml down

Or:

.. code-block:: bash

    docker-compose -f docker-compose-no-s3.yml down

The data will persist on your server at `/var/lib/mysql`, so you can safely stop the database without losing any data. However, if performing major administrative tasks, it is worth :ref:`backup` first.

.. _initialization:

Initialization
--------------

Once the database is up and running, we need to initialize the internal MySQL databases, tables, and users. To do this, we provide a script that will run the necessary commands to set up the database. This script is located in the `install_scripts/database/` directory of the Antelop repository.

From a local machine, with a local installation of Antelop, run:

.. code-block:: bash

	curl -O https://raw.githubusercontent.com/marcotripodi/Antelope/main/install_scripts/sql-init/experimenters.csv
	curl -O https://raw.githubusercontent.com/marcotripodi/Antelope/main/install_scripts/sql-init/init_db.py

You now need to edit the `experimenters.csv` file to add the details of the experimenters who will be using the database. The necessary fields are self-explanatory.

Now run the script:

.. code-block:: bash

	python init_db.py

You will be prompted for login credentials. The username in this case should be `root`, and the password should be the one you set in the `docker-compose.yml` file. The script will then create the necessary tables and users, and populate them with the experimenters you specified in the `experimenters.csv` file.

Note that this method is also used to update the experimenters in the database - this is done by simply running the script again with the updated `experimenters.csv` file. The script will automatically detect any changes and update the database accordingly.

Once you have some users in the `experimenters` table, you need to give them MySQL accounts and privileges to login to Antelop. See the :ref:`adding-users` section for more details on this.