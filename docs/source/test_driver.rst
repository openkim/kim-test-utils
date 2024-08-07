=======================
Main Computational Code
=======================

.. contents:: Table of Contents

The next step is to write the main computational code. It must be contained in a Python file named ``test_driver/test_driver.py``, 
although you may include as many other Python files as you wish for utility functions.

.. note::

  Your ``test_driver`` directory should be structured as a Python package. This means that it contains a (probably empty) ``__init__.py``,
  and you should use relative imports to import any other Python files from your ``test_driver.py``. For example, if you want to put
  some function ``my_helper_function`` in a Python file ``helper_functions.py``, you shoud structure your ``test_driver`` directory like this:

  ::

    test_driver/
    ├─ __init__.py
    ├─ test_driver.py
    ├─ helper_functions.py

  Then, in your ``test_driver.py`` you would import ``my_helper_function`` like this:

  .. code-block:: Python

    from .helper_functions import my_helper_function


You must create a class named ``TestDriver`` inheriting from  :class:`~kim_tools.CrystalGenomeTestDriver`.
In your ``TestDriver`` class, you must overload the function :func:`~kim_tools.KIMTestDriver._calculate`. 
Besides ``self``, the function must also accept ``**kwargs``. Before ``**kwargs``, you may add any additional arguments that 
you wish users or the OpenKIM Pipeline to be able to vary. 

An example ``test_driver.py`` from |example_url| is shown below. The ``_calculate`` function computes the energy vs. volume 
curve for isotropic expansion and compression of a crystal at zero temperature. You can use it as a starting point for your Test Driver. 
See the comments for explanations. Documentation regarding more complex usage of :mod:`kim_tools` can be found below the example.

Once ``test_driver.py`` is written, you will have to make some small modifications to the auxiliary files provided in the example 
Test Driver in order for your Driver to work in the OpenKIM pipeline: :doc:`tutorial_pipeline`. Before this, you will likely wish
to debug your code by invoking it directly in Python. See how to do that here: :doc:`tutorial_debug`

.. note::

  Temperature and stress are commonly used, so they are built-in attributes and do not need to be added as additional arguments: 

    *  :attr:`~kim_tools.CrystalGenomeTestDriver.temperature_K`
    *  :attr:`~kim_tools.CrystalGenomeTestDriver.cell_cauchy_stress_eV_angstrom3`

.. _doc.example_test_driver:

Example ``test_driver.py``
==========================

.. literalinclude:: ../../examples/CrystalGenomeASEExample__TD_000000654321_000/test_driver/test_driver.py
    :language: Python

Other functionality of :mod:`kim_tools`
============================================

LAMMPS and other non-ASE simulators
-----------------------------------
.. todo::

  As mentioned in the comments in the example above, non-ASE calculations require extra steps. 
  For example, one way to write run a LAMMPS simulation in this framework is to export your atomic configuration using :func:`ase.io.write`, create LAMMPS input file(s)
  with `kim commands <https://docs.lammps.org/kim_commands.html>`_ using the KIM model stored in the base class' attribute :attr:`~kim_tools.KIMTestDriver.kim_model_name`,
  run your simulation(s), and read the configuration back in using :func:`ase.io.read` (see below for why you may wish to read in a configuration after MD). In the future,
  we will integrate this functionality into :mod:`kim_tools`.

Structure Checking
------------------

For many simulations, such as the example above, there is no distinction between initial and final structures (other examples: phonons, elastic constants). Thus, when we invoke 
:func:`~kim_tools.CrystalGenomeTestDriver._add_property_instance_and_common_crystal_genome_keys`, the crystal description that is written to the property instance is the
same structure that the class was initialized with. However, for other types of simulations, such as MD or relaxation, the structure does change, and we should output the
changed structure. To update the crystal description from ``self.atoms`` or a different :class:`~ase.Atoms` object before writing, use ``self._update_crystal_genome_designation_from_atoms``:

.. autofunction:: kim_tools.CrystalGenomeTestDriver._update_crystal_genome_designation_from_atoms
  :noindex:

.. note::

  If the symmetry of your crystal has changed in the process of your simulation, this function will raise a :class:`~kim_tools.KIMTestDriverError`.
  This is intended and it is expected that you do not handle this exception. In the Crystal Genome framework, it is our convention that if the 
  crystal undergoes a symmetry-changing phase transition, the result is invalid and the Test Driver should exit with an error.

.. note::

  If you are running an MD simulation, the structure you report should be time-averaged and likely averaged over the supercell folded back into the unit cell. 
  This will give you more robust averages. Additionally, if you are performing an NPT simulation, you may as well write an instance of the 
  `crystal-structure-npt <https://openkim.org/properties/show/crystal-structure-npt>`_ property for future re-use. Note the optional ``restart-file`` key. 
  It is recommended that you save a restart file (for example, ``restart.dump``). You can then use the :func:`kim_tools.KIMTestDriver._add_file_to_current_property_instance`
  to add it to the property instance. It will automatically be numbered according to the current property instance and moved to the ``output`` directory if it is not already
  there (e.g. ``restart.dump`` will be automatically moved to the path ``output/restart-1.dump`` and reported in the property accordingly.)

  .. code-block:: Python

    self._add_property_instance_and_common_crystal_genome_keys("crystal-structure-npt",write_temp=True,write_stress=True)
    self._add_file_to_current_property_instance("restart-file","restart.dump")


.. todo::

  This should be an integrated part of ``kim-tools`` along with general ability to read and write LAMMPS files.

You may also find it useful to check for a symmetry change without changing the stored description of the crystal. This is useful, for example,
if your Test Driver is changing temperature or stress around some reference value, and you wish to make sure that the state changes have not
induced a phase transition. To do this, use ``self._get_crystal_genome_designation_from_atoms_and_verify_unchanged_symmetry`` (see below, it is likely that you can ignore the returned values). 
This function will also raise an exception, which you may or may not wish to catch (for example, if you are checking for phase transitions 
during a pressure scan, a phase transition likely indicates that you should limit the range of your scan, not discard the entire result).

.. autofunction:: kim_tools.CrystalGenomeTestDriver._get_crystal_genome_designation_from_atoms_and_verify_unchanged_symmetry
  :noindex: