.. SurrModel-Tutorial documentation master file, created by
   sphinx-quickstart on Wed Aug  8 14:43:57 2018.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

SurrModel-Tutorial
==================

This tutorial will go over the basics on how to create and use a surrogate
model to estimate the dynamic loading effects in a wind turbine blade optimization
routine.

Set-up
~~~~~~

To begin, clone this branch of AeroelasticSE_. AeroelasticSE is a python wrapper
for FAST. In addition, clone this branch of RotorSE_. RotorSE is an engineering
model for the analysis and optimization of wind turbine rotors.
Both AeroelasticSE and RotorSE are dependent on a number of sub-packages, so
make sure these are installed correctly.
Finally, compile
a FAST executable. Instructions on how to do this can be found in our previous
tutorial_.

.. _tutorial: https://fast-tutorial.readthedocs.io/en/latest/
.. _RotorSE: https://github.com/byuflowlab/RotorSE
.. _AeroelasticSE: https://github.com/byuflowlab/AeroelasticSE/tree/rotorse_fast_connection

Specifications to train surrogate model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Before we can train points to use in the surrogate model, we need to calculate
the wind turbine torque at rated wind speed.

Within the RotorSE directory, locate the script runOPT.py. In runOPT.py, specify::

  FASTinfo['calc_fixed_DEMs'] = True

Make sure that all the other options in that block are set to False.
In addition, set the
description variable to some appropriate string, such as calc_tq.

Within the RotorSE directory, locate the script FAST_util.py.
There are a few things that need to be specified in FAST_util.py. First,
make sure that all the checks in setupFAST_checks are set to False. In the
setupFAST_other function, set::

  FASTinfo['save_rated_torque'] = True

In the specify_DLCs function, set::

  DLC_List = ['DLC_0_0']

From the command line, navigate to the folder that contains RunOPT.py. Then, run::

  python RunOPT.py

This command should create a file that contains the torque at rated speed.
We can now set up the training point calculations. In runOPT.py, specify::

  FASTinfo['calc_surr_model'] = True

Make sure that all the other options in that block are set to False. Set the
description to some appropriate string, such as test_surrmodel.

There are a few things that need to be specified in FAST_util.py. First,
make sure that all the checks in setupFAST_checks are set to False.
Next, specify the locations of the virtual strain gages used in the FAST routine.
You can do this by setting::

  FASTinfo['sgp'] = [1, 2, 3]

This means that for each wind input file, FAST will run three times. The training
will take longer, but this way there won't be any interpolated data used to train
the surrogate model and will result in a more accurate surrogate model.

In the specify_DLCs function, make sure that::

  DLC_List = ['DLC_1_2', 'DLC_1_3', 'DLC_1_4','DLC_1_5', 'DLC_6_1', 'DLC_6_3']
  FASTinfo['rand_seeds'] = np.linspace(1, 6, 6)
  FASTinfo['mws'] = np.linspace(5, 23, 10)

when training a full surrogate.
However, for test purposes, you will want to use a small subset
of design load cases to make sure that it is working properly without taking too long.
An example of a test set is::

  DLC_List = ['DLC_1_2', 'DLC_1_3', 'DLC_6_1', 'DLC_6_3']
  FASTinfo['rand_seeds'] = np.linspace(1, 1, 1)
  FASTinfo['mws'] = np.linspace(11, 11, 1)

In the create_surr_model_params function, specify how many points we will attempt
to train in the surrogate model. For example, if we want to train 1000, points::

  FASTinfo['num_pts'] = 1000

Also, make sure that::

  FASTinfo['training_point_dist'] = 'lhs'

An example of 100 points being specified with linear hypercube spacing in two
directions is shown below.

.. image:: ./lhs.png


.. note:: Initial work supported a full factorial option (linear) as well as linear hypercube spacing (lhs), but functionality has only been developed for lhs.

In the setupFAST function, specify which reference turbine design template will be
used. For example, if we want to use the WindPACT 5.0 MW reference turbine::

  FASTinfo['FAST_template_name'] = 'WP_5.0MW'

Note that if a WindPACT reference turbine is used, also set::

  FASTinfo['set_chord_twist'] = True

We can now train the surrogate model. An example of doing this would be to
include the following lines in a batch script::

  #SBATCH --array=1-999 # job array size
  echo ${SLURM_ARRAY_TASK_ID}
  python runOPT.py ${SLURM_ARRAY_TASK_ID}

on the supercomputer. The size of
the array will depend on the number of training points. Once all the points
have been trained, use the function combine_results
(in the script FAST_Files/combine_sm_results.py). This is needed if only one wind turbine
is being used to train the surrogate model, or a number of wind turbines. Set
opt_file_srcs as the string descriptions defined in runOPT.py, and opt_file_dest
as the description used in runOPT.py when we next use the surrogate model. Note that
some data may not have been recorded because for some turbine designs, FAST may not
have been able to converge.
This function does have some nice functionality in that it can handle these cases.

Specifications to create surrogate model
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once the training data has been created, set::

  FASTinfo['opt_with_surr_model'] = True

and all the other options in the block as false. Also make sure that the string
set to description matches what was used for opt_file_dest previously.

In create_surr_model_params, set what type of approximation model you would like
to use. For example, if you want to use radial basis functions, set::

   FASTinfo['approximation_model'] = 'RBF'

In addition, if you desire to use a Kriging function, set the initial hyper-parameter
values::

  FASTinfo['theta0_val'] = [1e-2]

With these options set, when RunOPT.py is run again, it should create the surrogate
model before the optimization begins.

.. toctree::
   :maxdepth: 2
   :caption: Contents:



Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
