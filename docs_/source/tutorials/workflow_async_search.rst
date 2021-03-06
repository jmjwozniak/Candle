Run Asynchronous Search based hyperparameter optimization on CANDLE Benchmarks
==============================================================================

async-search is an asynchronous iterative optimizer written in Python.
It evaluates the best values of hyperparameters for CANDLE “Benchmarks”
available here: ``git@github.com:ECP-CANDLE/Benchmarks.git``

Running
-------

1. cd into the **Supervisor/workflows/async-search/test** directory
2. Specify the async-search parameters in the *cfg-prm-1.sh* file
   (INIT_SIZE, etc.).
3. Specify the PROCS, queue etc. in **cfg-sys-1.sh** file (NOTE:
   currently INIT_SIZE must be at least PROCS-2)
4. You will pass the MODEL_NAME, SITE, and optional experiment id
   arguments to **test-1.sh** file when launching:
   ``./test-1.sh <model_name> <machine_name> [expid]`` where
   ``model_name`` can be tc1 etc., ``machine_name`` can be local, cori,
   theta, titan etc. (see `NOTE <#making_changes>`__ below on creating
   new SITE files.)
5. The parameter space is defined in a **problem*.py** file (see
   **workflows/async-search/python/problem_tc1.py** for an example with
   tc1.). This is imported as ``problem`` in **async-search.py**.
6. The benchmark will be run for the number of processors specified
7. Final objective function values, along with parameters, will be
   available in the experiments directory and also printed

User requirements
-----------------

What you need to install to run the workflow:

-  This workflow - ``git@github.com:ECP-CANDLE/Supervisor.git`` . Clone
   and switch to the ``master`` branch. Then ``cd`` to
   ``workflows/async-search`` (the directory containing this README).
-  TC1 benchmark - ``git@github.com:ECP-CANDLE/Benchmarks.git`` . Clone
   and switch to the ``frameworks`` branch.
-  benchmark data - See the individual benchmarks README for obtaining
   the initial data

Python specific installation needed:

::

   conda install h5py
   conda install scikit-learn
   conda install pandas
   conda install mpi4py
   conda install -c conda-forge keras
   conda install -c conda-forge scikit-optimize

Calling sequence
----------------

Function calls:

::

   test-1.sh -> swift/workflow.sh ->

         (Async-search via EQPy)
         swift/workflow.swift <-> python/async-search.py

         (Benchmark)
         swift/workflow.swift -> obj_folder/obj_app.swift ->
         common/sh/model.sh -> common/python/model_runner.py -> 'calls Benchmark'

         (Results from Benchmark returned directly to Async-search)
         obj_folder/obj_app.swift -> python/async-search.py

Scheduling scripts:

::

   test-1.sh -> cfg-sys-1.sh ->
         common/sh/<machine_name> - module, scheduling, langs .sh files

Making Changes 
---------------

To create your own SITE files in workflows/common/sh/: - langs-SITE.sh -
langs-app-SITE.sh - modules-SITE.sh - sched-SITE.sh config

copy existing ones but modify the langs-SITE.sh file to define the EQPy
location (see workflows/common/sh/langs-local.sh for an example).

Structure
~~~~~~~~~

The point of the script structure is that it is easy to make copy and
modify the ``test-*.sh`` script, and the ``cfg-*.sh`` scripts. These can
be checked back into the repo for use by others. The ``test-*.sh``
script and the ``cfg-*.sh`` scripts should simply contain environment
variables that control how ``workflow.sh`` and ``workflow.swift``
operate.

``test-1.sh`` and ``cfg-{sys,prm}-1.sh`` should be unmodified for simple
testing.

The relevant parameters for the asynchronous search algorithm are
defined in ``cfg-*.sh`` scripts (see example in ``cfg-prm-1.sh``). These
are: - INIT_SIZE: The number of initial random samples. (Note: INIT_SIZE
needs to be larger than PROCS-2 for now.) - MAX_EVALS: The maximum
number of evaluations/tasks to perform. - NUM_BUFFER: The size of the
tasks buffer that should be maintained above the available workers
(num_workers) such that if the currently out tasks are less than (num
workers + NUM_BUFFER), more tasks are generated. - MAX_THRESHOLD: Under
normal circumstances, when a single model evaluation is finished, a new
hyper parameter set is produced for evaluation. If the model evaluations
occur within 15 seconds of each other, a MAX_THRESHOLD number of
evalutions must occur before the corresponding number of new values are
produced for evaluation. This can help with performance when many models
finish within a few seconds of each other. - N_JOBS: The number of jobs
to run in parallel when producing points (i.e. hyperparameter values)
for evaluation. -1 will set this to the number of cores.

Where to check for output
~~~~~~~~~~~~~~~~~~~~~~~~~

This includes error output.

When you run the test script, you will get a message about
``TURBINE_OUTPUT`` . This will be the main output directory for your
run.

-  On a local system, stdout/stderr for the workflow will go to your
   terminal.
-  On a scheduled system, stdout/stderr for the workflow will go to
   ``TURBINE_OUTPUT/output.txt``

The individual objective function (model) runs stdout/stderr go into
directories of the form:

``TURBINE_OUTPUT/EXPID/run/RUNID/model.log``

where ``EXPID`` is the user-provided experiment ID, and ``RUNID`` are
the various model runs generated by async-search, one per parameter set,
of the form ``R_I_J`` where ``R`` is the restart number, ``I`` is the
iteration number, and ``J`` is the sample within the iteration.
