#############
Abaqus on AWS
#############

.. contents::
    :backlinks: none
    :depth: 2


Information
===========
Abaqus is an FEA application used in manufacturing.

TL;DR
=====
* Use an OS with the 3.10 kernel
* Disable Hyperthreading
* Use Platform MPI or Intel MPI
* Set number of MPI threads to match core count
* Enable pinning of MPI to cores
* Set Clock Source to TSC

Scaling Behavior
================
Abaqus has a few different running modes with different hardware requirements.

I have a customer who's testing Abaqus on 72 cores using c4.8xlarge instances with the guidance below and is seing performance within ~15% of their on premise infiniband backed Haswell cluster

Performance Guidance
====================
Abaqus seems to be especially sensitive to the Linux kernel version.  For one customer, moving from a 2.6.32 Kernel to 3.10 saw the following performance improvements.

UC5 & UC4 represent the customer's model names.  Moving from a 2.6 kernel to 3.10 took the run time from 103 minutes to 78 minutes.

.. image:: abaqus_results.jpg

The customer was also able to set the number of MPI threads to match that of the number of cores.  When doing this, they took the run time even further down to 65 minutes.

Clock Source
------------
When monitoring the application, you can monitor the application to see calls it's making to the kernel

.. code-block:: none 

    strace -c -p <process id>

Let that run for ~15 seconds and hit Ctrl+C.

You'll likely see a number of calls related to gettimeofday.

Try setting the clock source to TSC and rerunning the application.

.. code-block:: none

    echo "tsc" > /sys/devices/system/clocksource/clocksource0/current_clocksource

Or, set edit grub.conf and add the following to the kernel line and reboot.

.. code-block:: none

    clocksource=tsc tsc=reliable

Abaqus Support Docs
===================
Here's a few tidbits from Abaqus support

    The Abaqus mp_process_binding parameter can be used to control the internal CPU binding option.   
    Setting this parameter to ON will force to Abaqus to utilize the internal binding option, when the analysis is not running on all the cores.  Careful attention must be paid to ensure that no other processes running on the same machine are also utilizing process binding. In particular multiple Abaqus analyses cannot be running on the same machine with this option enabled, or .severe performance degradation will result.  Multiple Abaqus analyses running on the same machine will bind and compete for the exact same cores leaving other core free.

    mp_process_binding=ON
     
    https://kb.dsxclient.3ds.com/mashup-ui/page/document?q=docid:QA00000020963
     
     
    I have set the above in the custom_v6.env on AWS cluster.  Please retest.    
    If the performance issues persist, please append the following to your Abaqus 2016 ~/site/custom_v6.env environment file and run the s2a benchmark job across multiple nodes. Please zip and send any job files

.. code-block:: none

    import os os.environ['ABA_RESOURCE_MONITOR']='yes'
    os.environ['ABA_RESOURCE_USEMALLINFO’]=”1”
    os.environ['ABA_GETMEMORYVALUE']='1'
    mp_mpirun_options="-v -d -T -prot"
    verbose=3

Implicit vs Explicit
====================
Abaqus/Explicit:
  - CPU intensive (higher clock speed & more cores are better)
  - I/O Intensive
  - MPI limited
  - Scales well up to hundreds of cores (< 500 cores)

Abaqus/Implicit:
  - Memory Intensive (more RAM the better)
  - Less I/O Intensive
  - MPI limited
  - Scales well up to 64-96 cores (< 100 cores)

