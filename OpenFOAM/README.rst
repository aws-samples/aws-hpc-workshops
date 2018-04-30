###############
OpenFOAM on AWS
###############

.. contents::
    :backlinks: none
    :depth: 2

****************************
CFD Direct AMI (Marketplace)
****************************

On Marketplace, **CFD Direct** has AMI with OpenFOAM installed and setup

Links
  http://cfd.direct/cloud

  https://aws.amazon.com/marketplace/pp/B017AHYO16/



****
Demo
****


Create the demo cluster
=======================

**Demo Config file**

.. code-block:: none

    [aws]
    aws_region_name = us-west-2
    aws_access_key_id = <your_access_key>
    aws_secret_access_key = <your_secret_access_key>

    [cluster default]
    vpc_settings = cluster-vpc
    key_name = <your_key_name>
    scheduler = sge
    base_os = centos6
    maintain_initial_size = true
    master_instance_type = c4.8xlarge
    compute_instance_type = c4.8xlarge
    initial_queue_size = 2
    min_queue_size = 2
    max_queue_size = 5
    encrypted_ephemeral = true
    pre_install = https://s3-us-west-2.amazonaws.com/cfncluster-public-scripts/disable_hyperthreading_preinstall
    post_install = https://s3-us-west-2.amazonaws.com/cfncluster-public-scripts/openfoamtutorial_centos.sh
    ebs_settings = custom

    [ebs custom]
    ebs_snapshot_id = snap-d7efa994

    [vpc cluster-vpc]
    vpc_id = <default_vpc_id>
    master_subnet_id = <subnet_id_from_chosen_AZ>

    [global]
    update_check = true
    sanity_check = true
    cluster_template = default


**Create the cluster**

.. code-block:: none

    cfncluster -c config_openfoam_demo create demo1


Setup environment
=================

.. code-block:: none

    sudo chown -R centos:centos /shared/

.. code-block:: none

    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/shared/OpenFOAM/mesa/lib


Submit OpenFOAM job
===================

Once the cluster has been created, login to the Master node and you should see two nodes available with qhost:

.. code-block:: none

    $ qhost
    HOSTNAME                ARCH         NCPU NSOC NCOR NTHR  LOAD  MEMTOT  MEMUSE  SWAPTO  SWAPUS
    ----------------------------------------------------------------------------------------------
    global                  -               -    -    -    -     -       -       -       -       -
    ip-172-31-44-135        lx-amd64       18    2   18   18  0.01   58.8G  603.1M     0.0     0.0
    ip-172-31-44-136        lx-amd64       18    2   18   18  0.01   58.8G  604.2M     0.0     0.0


Submit the job:

.. code-block:: none

    qsub ParallelBike.job

Wait for the job to complete, you should see ``Job Complete`` at the end of the output file.

Run visualization script
========================

.. code-block:: none

    ./RunVisualization.sh

You should see something similar to this at the bottom of the output:

.. code-block:: none

    View output at http://ec2-11-22-33-44.us-west-2.compute.amazonaws.com/output.png

Example output image:

.. image:: _images/OpenFOAM_output.png
    :width: 300px

|


