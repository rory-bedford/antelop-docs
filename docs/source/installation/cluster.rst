Cluster Installation
====================

Overview
--------

Antelop's cluster pipelines, which we refer to as *workflows*, are built primarily in `Nextflow <https://www.nextflow.io/>`_. Nextflow is a domain-specific language that allows the orchestration of complex high-performance computational pipelines. In particular, it allows us to run different jobs in different containers, easily configure resources for each component of a pipeline, and deploy our pipeline to a wide variety of computational infrastructures. In addition to this, we make heavy use of `Apptainer <https://apptainer.org/>`_ containers. 

Here we document the typical use-case of installing Antelop on a cluster in an academic institution. If you want to deploy on a local machine, or on a cloud provider, please `get in touch <mailto:rorybedford@protonmail.com>`_ with us, and we can help you set this up.

Prerequisites
-------------

To run Antelope workflows on a cluster, you'll need:

- **Linux Distribution**: A recent Linux distribution (Ubuntu 20.04+, Rocky Linux 8+, RHEL 8+, etc.)

  - Must support container technologies
  - Kernel version 4.8+ required

- **Container Runtime**: Either Apptainer (formerly Singularity) 1.0+ or Singularity 3.8+

  - `Apptainer Installation Guide <https://apptainer.org/docs/admin/main/installation.html>`_
  - `Singularity Installation Guide <https://docs.sylabs.io/guides/3.8/admin-guide/installation.html>`_
  - Must be configured to allow unprivileged users to run containers

- **Storage**:

  - **Scratch Space**: Fast, temporary storage (preferably SSD-backed)

    - Minimum 100GB per concurrent workflow
    - Used for temporary data processing and intermediate results
    - High I/O throughput recommended (>1GB/s)

  - **Long-term Storage**: Reliable storage for input data and results

    - Sufficient capacity for raw and processed data (typically TBs)
    - Accessible from compute nodes via a distributed filesystem (e.g., BeeGFS, Lustre, GPFS, CephFS)

  - **Public Script Storage** Publicly accessibly storage for the Antelop installation

    - This is where the Antelop installation will be stored, and where the scripts and containers will be pulled from
	- Needs to have execute permissions for the user running the workflowj:want

- **Job Scheduler**: Slurm Workload Manager

  - Version 24.00+ recommended
  - Properly configured partitions with appropriate resource limits
  - User permissions to submit jobs to relevant partitions

- **Network Connectivity**:

  - High-speed interconnect between compute nodes (10+ Gbps)
  - Internet access from login nodes for container pulls (or configured proxy)

- **Nextflow**:

  - Java 17+ installed and accessible
  - We install the Nextflow binary with the Antelop installation

These requirements are for standard cluster deployments. Specific workflows might have additional requirements detailed in their respective documentation.

Configuration
-------------

Prior to installing Antelop's workflows, you need to perform some configuration. The best thing to do it clone the Antelop repository on your cluster, as follows:

.. code-block:: bash

    git clone https://github.com/marcotripodi/Antelope

Within the repository, navigate to the `install_scripts/workflows/` directory. Here you will find two files that you need to edit:

- `workflows_config.toml`: This file contains the configuration for the Nextflow pipelines. In particular, you need to specify:

  - The location of the Antelop cluster installation (must be publicly accessible and executable)
  - The location where data results will get saved (can be user-specific or public)
  - The location of the scratch space for cluster intermediate data
  - The email address for job notifications (can include $USER)
  - The unix group if you want to limit institutional access to the workflows (can be all if you don't ahve a unix group)
  - What you want to install (set to all if it's your first time installing)

- `cluster_container_config.toml`: This file is identical to the `config.toml` used in your local Antelop installation, but points the containerised version of Antelop installed on the cluster to your database and storage. You need to set:

  - The location of the database (can be a public IP or hostname)
  - The location of the S3 storage (can be a public IP or hostname, or 'local')
  - Any GitHub repositories used by the lab for analysis scripts if you want to run analysis scripts on the cluster

Installation
------------

With these files configured you can now install Antelop on the cluster. To do this, you just need to run `install_workflows.py` in the `install_scripts/workflows/` directory. This will install all the workflows you specified in the `workflows_config.toml` file, and will build all the containers needed to run our workflows.

The only non-standard dependency to run this script is `toml`, which is now in the Python standard library as of Python 3.11. If you are using an older version of Python, make sure you pip install `toml` first.

Job Parameters
--------------

All of our Nextflow workflows trigger different SLURM processes on the cluster, which all request different resrouces, including clock wall time, memory, and CPU/GPU cores. We have set the defaults to be reasonable for most use cases - ie, spikesorting a 1 hour Neuropixels recording, running DeepLabCut inference on a 1 hour video, etc. However, it is possible that some of these jobs could fail due to not being assigned enough resources.

If this happens, you need to edit the resources yourself. We recommend doing this in your cloned repository so you can store the changes you've made yourself. Under `workflows-templates`, there is a directory for each workflow. Each of these directories contains a `nextflow.config` file, which contains the parameters for each process. You can edit these files to change the resources requested for each process. Note that a workflow is made of many different processes, so you may need to either read the logs from your failed to job to figure our which process failed, or edit multiple process in `nextflow.config` to get the job to run.

Resources are requested as standard SLURM command line options, and are specified under `clusterOptions` for that process, for example:

.. code-block:: bash

    clusterOptions = '-c 1 -t 60 -p cpu --mem=20G'

Requests just one CPU core, 60 minutes of wall time, the `cpu` partition, and 20GB of memory.

Alternate Installations
-----------------------

All of our workflows are written in Nextflow, because of Nextflow's fantastic ability to run on a wide variety of cluster systems. We distribute Antelop using SLURM by default due to its ubiquity in academic scientific computing, however, if your requriements differ (for example, if you are using a cloud provider such as AWS and want to use AWS Batch), we can easily adapt the configuration files to seamlessly integrate with your system. If you would like to do this, please `get in touch <mailto:rorybedford@protonmail.com>`_.