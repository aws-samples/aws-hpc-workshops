##########
WRF on AWS
##########

**Weather Research and Forecasting Model**

This guide documents the process of running WRF on AWS, including compiling WRF using the Intel compiler.

.. note::  This guide uses the trial version of the Intel compiler, if you plan on running this in production, please contact Intel for a license.

TL;DR

#. Setup and launch cluster with CfnCluster
#. NetCDF:  Download, build with Intel compiler, and install
#. WRF:  Download, build with Intel compiler, and install
#. Run CONUS 2.5k benchmark
#. View result with ncview


************
Introduction
************

**WRF is used for both research and operation forecasting.**

Main Web Site:  
  http://www.wrf-model.org

V3 User Guide:
  http://www2.mmm.ucar.edu/wrf/users/docs/user_guide_V3/contents.html

Introduction Presentation:
  WRF Modeling System Overview - http://www2.mmm.ucar.edu/wrf/users/tutorial/201601/overview.pdf

Tutorials
=========

Conference Tutorials:
    Many WRF tutorials can be found on Tutorials Presented at user conference:
    
    http://www2.mmm.ucar.edu/wrf/users/supports/tutorial.html

OnLine WRF Tutorial
    This online tutorial is a great place to start.  It walks through some of the basic WRF information, compiling and usage.

    http://www2.mmm.ucar.edu/wrf/OnLineTutorial/index.htm


*****************************
Cluster Setup with CfnCluster
*****************************

CfnCluster ( https://aws.amazon.com/hpc/cfncluster/ ) will be used to setup the cluster.

If this is your first time using CfnCluster, you should familiarize yourself with the usage.  Documentation can be found here: https://aws.amazon.com/hpc/cfncluster/ .  You should create a small test cluster before attempting to create the cluster you will use with your application.

You can use **Getting Started with CfnCluster** ( http://cfncluster.readthedocs.io/en/latest/getting_started.html ) to do the initial configuration.


Create a Post Install file specific to WRF
==========================================

As part of the cluster setup, you will need to create a post install file that will be used when each instance starts.  This post install script performs these operations on all instances (including the Master instance):

- Installs additional packages
- Disables HyperThreading
- Sets the clocksource to "tsc"
- Sets TCP values
- Sets ulimit values - limits need to be setup properly to run (e.g. stack size: ``ulimit -s``)

`Appendix C - Post Install Script`_ has the script.  Using this script, create a file in a S3 bucket similar to ``s3://bucket-id1-cfncluster/cfncluster_postinstall.sh``.  Use that file for the ``post_install`` argument in the CfnCluster config file, and you will also need to add the bucket to the ``s3_read_write_resource`` option:

For example:

.. code-block:: bash

    post_install = s3://bucket-id1-cfncluster/cfncluster_postinstall.sh
    s3_read_write_resource = arn:aws:s3:::bucket-id1-cfncluster/*


Once you have CfnCluster installed, create the cluster with the additional options below.  These options are added or replace options to the previously created ``~/.cfncluster/config`` file.  Many of the CfnCluster settings can use the default values (i.e. don't need to be included in the config file).  These are cluster settings that have yielded positive results for WRF.  The instance type chosen should not be considered the only one that works, but for this guide the ``c4.8xlarge`` instance type will be used.

.. warning::  Several of these settings will result in higher cost.  Please review the `EC2 costs <https://aws.amazon.com/ec2/pricing/>`__  prior to cluster creation.

.. code-block:: bash

    [cluster wrf]
    compute_instance_type = c4.8xlarge
    master_instance_type = c4.8xlarge
    master_root_volume_size = 100
    cluster_type = ondemand
    placement = cluster
    placement_group = DYNAMIC
    base_os = alinux
    extra_json = { "cfncluster" : { "cfn_scheduler_slots" : "cores" } }
    s3_read_write_resource = arn:aws:s3:::bucket-id1-cfncluster/*
    post_install = s3://bucket-id1-cfncluster/cfncluster_postinstall.sh
    ebs_settings = wrf-ebs

    [ebs wrf-ebs]  ## Used for the NFS mounted file system
    volume_type = io1
    volume_size = 250
    volume_iops = 5000


Create the cluster
==================

After creating the post install script, and setting options in the CfnCluster config file specific to your application, create the cluster.


.. note:: The remaining steps assume that you have created a cluster, and you can login to the Master instance.

***************************************
Download and install the Intel compiler
***************************************

Before building WRF or other related packages, the Intel compiler will need to be installed to achieve expected performance.  You can download the compiler here:

https://software.intel.com/en-us/intel-parallel-studio-xe

After the cluster has been created, login to the Master instance.  The Intel compiler needs to be installed in ``/shared`` on the **Master Instance**.



*************************
Download and build NetCDF
*************************

It is **strongly recommended** that you use NetCDF version 4.1.3 from the WRF compile link.

`Appendix B - Build NetCDF with the Intel compiler`_ has the link to NetCDF and the build instructions using the Intel compiler.


*****************************
Download WRF and weather data
*****************************

Download WRF - Requires Account
===============================

To be able to download WRF you will need an account on the WRF site.

#. Go to this page:

   http://www2.mmm.ucar.edu/wrf/users/download/get_source.html

#. Click on **New Users** (or **Retuning Users** if you already have an account)

#. Complete the registration or just enter in your email address

#. You should land on the download page, download **WRF-ARW**, the file should look something like this: http://www2.mmm.ucar.edu/wrf/src/WRFV3.8.1.TAR.gz

#. Optionally download **WPS**


Download weather data (as needed)
=================================

Data download - NCAR's RDA ( http://www2.mmm.ucar.edu/wrf/users/ ) (Research Data Archive)


***********
Compile WRF
***********

These steps should be done on the **Master Instance**.

These steps summarize the official steps, and use Intel compiler options.  Although, the official WRF guide uses the GNU compiler, you should see better performance with the Intel compiler.  Here is the official **Compile Tutorial** http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compilation_tutorial.php, and here is a **Compile WRF & WPS** http://www2.mmm.ucar.edu/wrf/users/tutorial/201601/compiling.pdf presentation.

.. note:: Building WRF using the Intel compiler

Setup directories and download WRF
==================================

Links to WRF code may be different, check the WRF site.  This assumes that you already have a WRF account.

.. code-block:: none

    $ cd /shared
    $ mkdir WRF
    $ cd WRF
    $ wget http://www2.mmm.ucar.edu/wrf/src/WRFV3.8.1.TAR.gz
    $ tar xvf WRFV3.8.1.TAR.gz
    $ cd WRFV3


Setup build env
===============

.. code-block:: none

    . /shared/intel/bin/compilervars.sh intel64
    export NETCDF=/shared/netcdf
    export WRFIO_NCD_LARGE_FILE_SUPPORT=1


Run "configure"
===============

Choose option "21" (SNB with AVX mods), and then option "1" for nesting:


.. code-block:: none

    $ ./configure
    checking for perl5... no
    checking for perl... found /usr/bin/perl (perl)
    Will use NETCDF in dir: /shared/netcdf
    HDF5 not set in environment. Will configure WRF for use without.
    PHDF5 not set in environment. Will configure WRF for use without.
    Will use 'time' to report timing information
    $JASPERLIB or $JASPERINC not found in environment, configuring to build without grib2 I/O...
    ------------------------------------------------------------------------
    Please select from among the following Linux x86_64 options:

      1. (serial)   2. (smpar)   3. (dmpar)   4. (dm+sm)   PGI (pgf90/gcc)
      5. (serial)   6. (smpar)   7. (dmpar)   8. (dm+sm)   PGI (pgf90/pgcc): SGI MPT
      9. (serial)  10. (smpar)  11. (dmpar)  12. (dm+sm)   PGI (pgf90/gcc): PGI accelerator
     13. (serial)  14. (smpar)  15. (dmpar)  16. (dm+sm)   INTEL (ifort/icc)
                                             17. (dm+sm)   INTEL (ifort/icc): Xeon Phi (MIC architecture)
     18. (serial)  19. (smpar)  20. (dmpar)  21. (dm+sm)   INTEL (ifort/icc): Xeon (SNB with AVX mods)
     22. (serial)  23. (smpar)  24. (dmpar)  25. (dm+sm)   INTEL (ifort/icc): SGI MPT
     26. (serial)  27. (smpar)  28. (dmpar)  29. (dm+sm)   INTEL (ifort/icc): IBM POE
     30. (serial)               31. (dmpar)                PATHSCALE (pathf90/pathcc)
     32. (serial)  33. (smpar)  34. (dmpar)  35. (dm+sm)   GNU (gfortran/gcc)
     36. (serial)  37. (smpar)  38. (dmpar)  39. (dm+sm)   IBM (xlf90_r/cc_r)
     40. (serial)  41. (smpar)  42. (dmpar)  43. (dm+sm)   PGI (ftn/gcc): Cray XC CLE
     44. (serial)  45. (smpar)  46. (dmpar)  47. (dm+sm)   CRAY CCE (ftn/cc): Cray XE and XC
     48. (serial)  49. (smpar)  50. (dmpar)  51. (dm+sm)   INTEL (ftn/icc): Cray XC
     52. (serial)  53. (smpar)  54. (dmpar)  55. (dm+sm)   PGI (pgf90/pgcc)
     56. (serial)  57. (smpar)  58. (dmpar)  59. (dm+sm)   PGI (pgf90/gcc): -f90=pgf90
     60. (serial)  61. (smpar)  62. (dmpar)  63. (dm+sm)   PGI (pgf90/pgcc): -f90=pgf90
     64. (serial)  65. (smpar)  66. (dmpar)  67. (dm+sm)   INTEL (ifort/icc): HSW/BDW
     68. (serial)  69. (smpar)  70. (dmpar)  71. (dm+sm)   INTEL (ifort/icc): KNL MIC

    Enter selection [1-71] : 21

    ------------------------------------------------------------------------
    Compile for nesting? (1=basic, 2=preset moves, 3=vortex following) [default 1]: 1

    Configuration successful!
    ------------------------------------------------------------------------
    testing for MPI_Comm_f2c and MPI_Comm_c2f
       MPI_Comm_f2c and MPI_Comm_c2f are supported
    testing for MPI_Init_thread
       MPI_Init_thread is supported
    testing for fseeko and fseeko64
    fseeko64 is supported
    ------------------------------------------------------------------------

    ... snip ...

    Testing for NetCDF, C and Fortran compiler

    This installation of NetCDF is 64-bit
                     C compiler is 64-bit
               Fortran compiler is 64-bit
                  It will build in 64-bit


Edit configure.wrf
==================

Make these changes:

.. code-block:: none

    DM_FC           =       mpiifort
    ...
    OPTAVX          =       -xHost
    CFLAGS_LOCAL    =       -w -O3 $(OPTAVX) -qopenmp
    ...
    FCOPTIM         =       -O3 $(OPTAVX) -qopenmp


Compile
=======

The compile time varies, but should take less than an hour.

.. code-block:: none

    $ ./compile em_real 2>&1 | tee compile.log

You should see four binaries successfully built, the output will also show the time taken for the compile:

.. code-block:: none

    ==========================================================================
    build started:   Fri Nov 18 18:31:37 UTC 2016
    build completed: Fri Nov 18 19:10:59 UTC 2016

    --->                  Executables successfully built                  <---

    -rwxrwxr-x 1 ec2-user ec2-user 46732186 Nov 18 19:10 main/ndown.exe
    -rwxrwxr-x 1 ec2-user ec2-user 46722859 Nov 18 19:10 main/real.exe
    -rwxrwxr-x 1 ec2-user ec2-user 45994883 Nov 18 19:10 main/tc.exe
    -rwxrwxr-x 1 ec2-user ec2-user 51485118 Nov 18 19:09 main/wrf.exe

    ==========================================================================


Run wrf.exe to verify build
===========================

Set limits
----------

The ``post_install`` script included in the appendix and mentioned above will set the necessary limits.  If you don't set the ``stack size`` to ``unlimited``, you will receive an error similar to this:

.. code-block:: none

    forrtl: severe (174): SIGSEGV, segmentation fault occurred

You can just set stack size, but you should set all limits with the instructions mentioned above as part of the cluster creation.

Set the stack size with this command:

.. code-block:: none

    ulimit -s unlimited


Run wrf.exe
-----------

Run ``wrf.exe`` at the command line, you will need to export the path of NetCDF and the Intel libraries:

.. code-block:: none

    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/shared/netcdf/lib:/shared/intel/lib
    $ ./wrf.exe
     starting wrf task            0  of            1

Check ``rsl.out.0000`` and ``rsl.err.0000`` for errors.



****************************
Run WRF CONUS 2.5k Benchmark
****************************

.. note:: This Benchmark run is specific to the c4.8xlarge instance type.  If you use another instance type, you will need to adjust the mpirun command and OpenMP threads.

The WRF CONUS 2.5k Benchmark is here:

http://www2.mmm.ucar.edu/wrf/WG2/benchv3/#_Toc212961289

Benchmark description from http://www2.mmm.ucar.edu/wrf/WG2/benchv3/#_Toc212961289 :

    "Single domain, large size. 2.5 km CONUS,  June 4, 2005"

    "Description: Latter 3 hours of a 9-hour, 2.5km resolution case covering the Continental U.S. (CONUS) domain June 4, 2005 with a 15 second time step.  The benchmark period is hours 6-9 (3 hours), starting from a restart file from the end of the initial 6 hour period. As an alternative, the model may be run 9 hours from cold start. "


Setup Benchmark
===============

Create the benchmark directory
------------------------------

It's important to use the ``-a`` flag with the copy command, it preserves all of the symbolic links to the WRF binaries.

.. code-block:: none

    $ cd WRFV3/test
    $ cp -a em_real em_real_2.5k_CONUS

Copy benchmark files
--------------------

The three files that were created above (restart file, boundary file, ``namelist.input`` file), need to be copied in to the ``WRFV3/test/em_real_2.5k_CONUS`` directory.


Files need for benchmark
========================

Before downloading or editing ``namelist.input``, move the original out of the way:

.. code-block:: none

    $ mv namelist.input namelist.input.dist

You will need three files to run the benchmark:

- Restart file (e.g. ``wrfrst_d01_2005-06-04_06_00_00``)
- Boundary file (e.g.  ``wrfbdy_d01``)
- ``namelist.input`` file


Option 1: Download files already prepared
-----------------------------------------

If you prefer, all three of these files have been created and can be downloaded here:

- https://s3.amazonaws.com/duff-public/wrf/2.5k_bench/namelist.input
- https://s3.amazonaws.com/duff-public/wrf/2.5k_bench/wrfbdy_d01  (284MB)
- https://s3.amazonaws.com/duff-public/wrf/2.5k_bench/wrfrst_d01_2005-06-04_06_00_00 (17GB)


Option 2: Follow steps on WRF site to construct files
-----------------------------------------------------

Go to the WRF CONUS 2.5k Benchmark site ( http://www2.mmm.ucar.edu/wrf/WG2/benchv3/#_Toc212961289 ), and follow the steps to download and construct the files needed.  If you do manually download and construct the files you will need to make these changes to the **namelist.input** file.

**Edit the namelist.input file:**

- Remove pNetCDF usage:

  The version of WRF used in this guide does not include using pNetCDF, so you will need to edit the ``namelist.input`` to reflect that.  In other words, "change the io_form_* settings in the time_control section of the namelist.input file from 11 to 2".

- Add ``use_baseparam_fr_nml = .t.`` to the ``&dynamics`` section:

  It should look like this:

  .. code-block:: none

      &dynamics
      w_damping                           = 1,
      diff_opt                            = 1,
      km_opt                              = 4,
      khdif                               = 0,
      kvdif                               = 0,
      non_hydrostatic                     = .true.,
      use_baseparam_fr_nml                = .t.,
      /

  Otherwise, you will see this error:

  .. code-block:: none

      -------------- FATAL CALLED ---------------
      FATAL CALLED FROM FILE:  start_em.b  LINE:     551
      start_em: did not find base state parameters in wrfinput. Add use_baseparam_fr_nml = .t. in &dynamics and rerun



Run at the command line
=======================

This assumes that the instances have two processors, each with 9 cores, running one task per processor each with 9 threads.

This shows a run with 1440 threads (OMP_NUM_THREADS * np, or for this case 9 * 160)

.. code-block:: none

    $ . /shared/intel/bin/compilervars.sh intel64
    $ export LD_LIBRARY_PATH=/shared/netcdf/lib:$LD_LIBRARY_PATH
    $ export OMP_NUM_THREADS=9
    $ export KMP_STACKSIZE=128M
    $ export KMP_AFFINITY=granularity=fine,compact,1,0
    $ qhost | grep ip- | awk {'print $1'} > ~/hostfile.80
    $ mpirun -hostfile ~/hostfile.80 -np 160 -ppn 2 ./wrf.exe

The KMP_AFFINITY variable is explained in detail here, in the *"permute and offset combinations"* section: https://software.intel.com/en-us/node/522691#PERMUTE_AND_OFFSET_COMBINATIONS_WITH_TYPE

   Short Description:
      "The OpenMP* thread n+1 is bound to a thread context as close as possible to OpenMP* thread n, but on a different core. Once each core has been assigned one OpenMP* thread, the subsequent OpenMP* threads are assigned to the available cores in the same order, but they are assigned on different thread contexts."

You should see this at the command line (for example):

.. code-block:: none

    $ mpirun -hostfile hosts.2 -np 4 -ppn 2 ./wrf.exe
    starting wrf task            1  of            4
    starting wrf task            2  of            4
    starting wrf task            0  of            4
    starting wrf task            3  of            4

Check the progress in the ``rsl.error.0000`` file:

.. code-block:: none

    $ tail -1000f rsl.error.0000


Verify wrf run
==============

You should see **SUCCESS COMPLETE WRF** at the bottom of the rsl.out.0000 file or on STDOUT (for serial):

.. code-block:: none

    d01 2001-10-25_03:00:00 wrf: SUCCESS COMPLETE WRF


Check timing with stats.awk
===========================

Download the ``stats.awk`` file from the WRF site:

    http://www2.mmm.ucar.edu/wrf/WG2/benchv3/stats.awk

Then use this command to gather the timing information:

.. code-block:: none

    $ grep 'Timing for main' rsl.error.0000 | tail -149 | awk '{print $9}' | awk -f stats.awk

Example output:

.. code-block:: none

    $ grep 'Timing for main' rsl.error.0000 | tail -149 | awk '{print $9}' | awk -f stats.awk
    ---
        items:       149
          max:         0.567450
          min:         0.154650
          sum:        27.599280
         mean:         0.185230
     mean/max:         0.326425


Viewing results with ncview
===========================

Install XWindows software on local machine.  For OSX, this is XQuartz ( https://www.xquartz.org/ )

Install ncview, xterm, and xauth on the **Master Instance**

.. code-block:: none

    sudo yum install ncview xterm xauth

Reconnect to the **Master Instance** with X forwarding and test with ``xterm``, and you should see it display on your desktop:

.. code-block:: none

    $ ssh -X -A -i key_1.pem ec2-user@<ip_address>

    [ec2-user@ip-address]$ xterm

Run ncview on the wrfout* file:

.. code-block:: none

    [ec2-user@ip-address]$ cd WRFV3/test/em_real_2.5k_CONUS
    [ec2-user@ip-address]$ ncview wrfout_d01_2005-06-04_09_00_00


You see the main application panel, and if you select **2d Vars -> UST** and click on the image, you should see something like this:

.. image:: _images/ncview_wrf.png
    :width: 800px







*************************************************
Appendix A - Build NetCDF with the Intel compiler
*************************************************

.. note::  Building NetCDF with the Intel compiler

The steps here summarize Intel's instructions from thier build notes: http://tinyurl.com/zvg7478

Download NetCDF 4.1.3 from the WRF Compile Tutorial site:
  http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/netcdf-4.1.3.tar.gz


Setup env vars
==============

Assumes compiler install in ``/shared/intel``:

.. code-block:: none

    $ cat netcdf.intel.env
    export PATH=$PATH:/shared/intel/bin
    export CC=icc
    export CXX=icpc
    export CFLAGS='-O3 -xHost -ip -no-prec-div -static-intel'
    export CXXFLAGS='-O3 -xHost -ip -no-prec-div -static-intel'
    export F77=ifort
    export FC=ifort
    export F90=ifort
    export FFLAGS='-O3 -xHost -ip -no-prec-div -static-intel'
    export CPP='icc -E'
    export CXXCPP='icpc -E'


Build
=====

.. code-block:: none

    $ . netcdf.intel.env
    $ tar xf netcdf-4.1.3.tar.gz
    $ cd netcdf-4.1.3
    $ . /shared/intel/bin/compilervars.sh ia32
    $ . /shared/intel/bin/compilervars.sh intel64
    $ ./configure --prefix=/shared/netcdf --disable-netcdf-4 --disable-dap
    $ make



Run "make check"
================

.. code-block:: none

    $ make check 2>&1 | tee make.check.out

You should see several ``All N tests passed`` messages in the ``make.check.out`` file:

.. code-block:: none

    $ grep passed make.check.out
    All 3 tests passed
    All 9 tests passed
    1 test passed
    1 test passed
    *** All tests of ncgen and ncdump using test0.cdl passed!
    *** All ncgen and ncdump with 64-bit offset format tests passed!
    *** All ncgen and ncdump test output for classic format passed!
    *** All ncgen and ncdump test output for 64-bit offset format passed!
    *** All ncdump test output for -t option with CF calendar atts passed!
    *** All utf8 tests of ncgen and ncdump passed!
    *** All nccopy tests passed!
    All 11 tests passed
    All 5 tests passed
    *** All tests of C++ API test output passed!
    All 6 tests passed
    All 2 tests passed
    All 7 tests passed
    All 7 tests passed
    All 7 tests passed



Run "make install"
==================

.. code-block:: none

    $ make install

    ... snip ...

    +-------------------------------------------------------------+
    | Congratulations! You have successfully installed netCDF!    |
    |                                                             |
    | You can use script "nc-config" to find out the relevant     |
    | compiler options to build your application. Enter           |
    |                                                             |
    |     nc-config --help                                        |
    |                                                             |
    | for additional information.                                 |
    |                                                             |
    | CAUTION:                                                    |
    |                                                             |
    | If you have not already run "make check", then we strongly  |
    | recommend you do so. It does not take very long.            |
    |                                                             |
    | Before using netCDF to store important data, test your      |
    | build with "make check".                                    |
    |                                                             |
    | NetCDF is tested nightly on many platforms at Unidata       |
    | but your platform is probably different in some ways.       |
    |                                                             |
    | If any tests fail, please see the netCDF web site:          |
    | http://www.unidata.ucar.edu/software/netcdf/                |
    |                                                             |
    | NetCDF is developed and maintained at the Unidata Program   |
    | Center. Unidata provides a broad array of data and software |
    | tools for use in geoscience education and research.         |
    | http://www.unidata.ucar.edu                                 |
    +-------------------------------------------------------------+


********************************
Appendix B - Post Install Script
********************************

.. code-block:: none

    #!/bin/bash

    USER=ec2-user

    # extra packages
    yum -y install screen dstat htop strace perf pdsh

    # Download and install hyperthread disabling script
    wget -O /etc/init.d/disable_hyperthreading https://cfncluster-public-scripts.s3.amazonaws.com/disable_hyperthreading
    chmod a+x /etc/init.d/disable_hyperthreading
    chkconfig --add /etc/init.d/disable_hyperthreading
    chkconfig --level 2345 disable_hyperthreading on
    /etc/init.d/disable_hyperthreading start

    # Switch the clock source to TSC
    echo "tsc" > /sys/devices/system/clocksource/clocksource0/current_clocksource

    # Set TCP windows
    cat >>/etc/sysctl.conf << EOF
    net.core.netdev_max_backlog   = 1000000

    net.core.rmem_default = 124928
    net.core.rmem_max     = 67108864
    net.core.wmem_default = 124928
    net.core.wmem_max     = 67108864

    net.ipv4.tcp_keepalive_time   = 1800
    net.ipv4.tcp_mem      = 12184608        16246144        24369216
    net.ipv4.tcp_rmem     = 4194304 8388608 67108864
    net.ipv4.tcp_syn_retries      = 5
    net.ipv4.tcp_wmem     = 4194304 8388608 67108864
    EOF

    sysctl -p

    # Set ulimits
    cat >>/etc/security/limits.conf << EOF
    # core file size (blocks, -c) 0
    *           hard    core           0
    *           soft    core           0

    # data seg size (kbytes, -d) unlimited
    *           hard    data           unlimited
    *           soft    data           unlimited

    # scheduling priority (-e) 0
    *           hard    priority       0
    *           soft    priority       0

    # file size (blocks, -f) unlimited
    *           hard    fsize          unlimited
    *           soft    fsize          unlimited

    # pending signals (-i) 256273
    *           hard    sigpending     1015390
    *           soft    sigpending     1015390

    # max locked memory (kbytes, -l) unlimited
    *           hard    memlock        unlimited
    *           soft    memlock        unlimited

    # open files (-n) 1024
    *           hard    nofile         65536
    *           soft    nofile         65536

    # POSIX message queues (bytes, -q) 819200
    *           hard    msgqueue       819200
    *           soft    msgqueue       819200

    # real-time priority (-r) 0
    *           hard    rtprio         0
    *           soft    rtprio         0

    # stack size (kbytes, -s) unlimited
    *           hard    stack          unlimited
    *           soft    stack          unlimited

    # cpu time (seconds, -t) unlimited
    *           hard    cpu            unlimited
    *           soft    cpu            unlimited

    # max user processes (-u) 1024
    *           soft    nproc          16384
    *           hard    nproc          16384

    # file locks (-x) unlimited
    *           hard    locks          unlimited
    *           soft    locks          unlimited
    EOF



