Administration
==============

.. _adding-users:

Adding users and privileges
---------------------------

Within Antelop, each user in the `experimenter` table must additionally be a user in the MySQL database. Furthermore, their username should be their institutional username, so that this can be used to perform operations via ssh.

To perform the following administrative operations, you need to have root credentials for the MySQL database, and a `MySQL client <https://www.mysql.com/>`_ installed on your machine.

First, you need to connect to the MySQL database. You can do this by running the following command in your terminal:

.. code-block:: bash

	mysql -u root -h hostname -p

To add a new user to the MySQL database, you can use the following command:

.. code-block:: sql

	CREATE USER 'username'@'%' IDENTIFIED BY 'password';

This will create a new user with the specified username and password. You can then grant them privileges to access the database with the following command:

.. code-block:: sql

	GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'username'@'%';

If you have collaborators with whom you want to share data, but don't want to be able to edit data, you can grant them jsut `SELECT` privileges:

.. code-block:: sql

	GRANT SELECT ON *.* TO 'username'@'%';

For any user to be able to use Antelop, they additionally need to have a corresponding entry in the `experimenter` table. This can be done as in :ref:`initialization`. Just add their entry to the `experimenters.csv` file, and run the `init_db.py` script again. This will automatically add them to the `experimenter` table, and update their privileges in the MySQL database.

.. _backup:

Making a backup
---------------

Before performing any major administrative tasks, it is worth making a backup of the database. This can be done using the `mysqldump` command. To do this, run the following command in your terminal:

.. code-block:: bash

	mysqldump -u root -h hostname -p --all-databases > backup.sql

If anything goes wrong, you can restore the database from the backup with the following command:

.. code-block:: bash

	mysql -u root -h hostname -p < backup.sql

This will restore the database from the backup file. Note that this will overwrite any existing data in the database, so be careful when using this command.

Regular backups
---------------

If you want to make regular backups of the database, you can use a cron job to automate this process. To do this, open the crontab file on an appropriate machine with the following command:

.. code-block:: bash

	crontab -e

Then add the following line to the file:

.. code-block:: bash

	0 0 * * * mysqldump -u root -h hostname -p --all-databases > /path/to/backup.sql

This will run the `mysqldump` command every day at midnight and save the backup file to the specified path. You can adjust the schedule as needed, and can save several backup files simultaneously by changing the filename to include the date, e.g. `backup-$(date +\%Y-\%m-\%d).sql`.

