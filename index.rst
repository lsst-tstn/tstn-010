
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: remove hard wrapping (and let the editor use soft-wrapping)

.. _ts_standardscripts: https://github.com/lsst-ts/ts_standardscripts
.. _ts_externalscripts: https://github.com/lsst-ts/ts_externalscripts
.. _ts_observing_utilities: https://github.com/lsst-ts/ts_observing_utilities
.. _ts_notebooks: https://github.com/lsst-ts/ts_notebooks
.. _ts_observatory_control: https://github.com/lsst-ts/ts_observatory_control

.. note::

    This technote is to detail out the observing scripts and Jupyter notebook development cycle from a simple test or idea, developed inside a notebook, to evolving into a method that could be called by other notebooks, and/or into a SAL Script to be called via the scriptQueue, then finally to the level of being a sanctioned and regularly maintained Script as part of operations.


Introduction
===============

The Rubin Observatory control system has been designed such that it enables astronomers and developers a great deal of flexibility when it comes to developing strategies for testing, system verification, commissioning, engineering and observations.
Nevertheless, in order to take full advantage of this flexibility a process needs to be in place to allow the incorporation of these processes back into the observatory sanctioned mainstream code base.
This technical note details such process from the initial exploratory phase all the way to a final sanctioned product.

Documented below are the details for each step in the development process, including which repos should be used for the different aspects of development as well as the level of testing and peer-review required at each stage.
The technote is built upon a sample use-case encountered early on in the commissioning phase of the Auxiliary Telescope.
In this use-case, the user wants to derive a high-level piece of functionality requiring new additions to software at multiple levels to enable the implementation of a final production SAL Script.
A top-level overview of the use-case is as follows:

    - Slew to a target -- Using existing functionality in the ``ATCS`` class
    - Take an image -- Using existing functionality in the ``LATISS`` class
    - Perform basic instrument-signature-removal (ISR) such that it can be analyzed, notably bias
      subtraction and cosmic-ray rejection/interpolation. -- Functionality exists in the science pipeline code but requires configuration for this application
    - Find the brightest point source in the image -- This functionality exists in the science pipeline code but requires configuration for this application
    - Calculate the telescope offset required to put the star on a specific pixel (within a tolerance) -- Required new functionality
    - Perform a series of observations using multiple instrument setups and exposure times. -- Requires new functionality to change the telescope focus (and pointing) for each filter/grating configuration.

A general overview of the development flow, as viewed from the user, is as follows:

    - Draft, test and flush-out their desired functionality in a Jupyter notebook.

      This may include creating drafts of functions (e.g. calculating offsets), performing calls to high-level classes
      (e.g. slew telescope) etc.

    - Create observing utilities to perform specific tasks, which may be sufficiently generic such that they may be used
      by other use-cases (e.g. the function of finding a star and calculating the offset to pixel [x,y]).

    - Create a SAL Script, which is runnable by the scriptQueue, to perform these tasks.

      Note that at this point the utilities may still be rough, and certain functionality might be better accomplished
      in lower classes (e.g. ATCS) but that functionality does not yet exist.

    - Request new functionality in lower-level control classes (e.g. ATCS)

    - Migrate/evolve the utilities and SAL Script to a production level for regular use with the scriptQueue

How the workflow moves through the various repositories is represented by the following diagram, where each of the
sections is discussed in detail below.

.. figure:: _static/Notebook_and_script_workflow_v2.jpg
    :width: 600px
    :align: center
    :alt: Workflow Summary

    A visual overview of the workflow process, starting from an original idea first demonstrated in a Jupyter notebook.

The three-tiered development strategy is being adopted to facilitate maximum flexibility to implement changes on short timescales.
The staging and development areas gives users full range to simultaneously operate across multiple versions/branches with unhindered flexibility.
This is especially important for commissioning activities where rapid code changes are required and temporary implementation is required which will break other people's code in the same area.

The production area is to provide a single code base where all code in that area (on the master and develop branches) is fully functional at all times.
This will be most important during operations when the codebase will be less dynamic but will also be useful for different commissioning teams that will circle through and want an established and stable repository to start from without having to diagnose what changes the previous group has made to facilitate their goals.
This area of the codebase will also require larger amounts of testing in order to ensure that changes made in one area do not result in breaking code in another.


.. _notebooks:

Jupyter Notebooks
=================
Jupyter notebooks (henceforth referred to as notebooks) will be the primary tool used in system verification and commissioning.
The use of them is not strictly required, however the environment permits the simultaneous control of observatory functionality, data reduction/analysis tasks and documentation and is supported by the project.
This is the natural starting point for development of ideas and demonstrating proof of concept(s).
In the use-case referenced in this technote, notebooks are the starting point, where the user is free to do as they wish with its structure/content etc.

.. Important::
    Notebooks are *not* to hold functional code over extended periods of time (~2 weeks) nor are they meant to augment observatory control software.
    If a piece of code (e.g. a function) developed in a notebook and is useful then it must be moved into a function in the development repositories discussed below.

A general rule of thumb is that if one finds themselves copying/pasting code from a notebook to another, then that code should not be in a notebook!
It is expected that if something is developed during a commissioning activity or observing run that this function be moved in short order.
If one does not have the know-how to do this then ask for assistance from other observatory personnel.

User's notebooks are currently stored in the `ts_notebooks`_ repository.
There is also a section where individuals create directories with their identifying username (e.g. pingraham or tribeiro).
Notebooks should be cleared of all data prior to committing/pushing, to prevent the repo size from rapid expansion in physical disk usage.
The repo also holds a series of `examples` which ranges from telescope operation to EFD mining/analysis.

Users should still follow the T&S development guidelines when using this repo.
That means, create a ticket branch to work on, commit code and, once ready, open a pull-request to have their work integrated to the `develop` branch.
Content added to the users directory are still subjected to the pull-request process but only to guarantee that the content was cleared out and that no changes where made to other users content (without permission).
Contents in the `examples` directory will be subject to a more rigorous review process and will require continuous integration (CI) testing upon availability.

.. note::

    A solution to implementing CI testing for notebooks is in development.
    This section will be updated upon release of the CI methodology.


.. note::
    If one is developing on the NCSA teststand, the mocking CSC or control class functionality may be required to perform tests with real data.
    Mocks are also useful in other aspects of development.
    Mocks are not simulators, but are generally empty classes/functions such that development can occur without causing import errors from libraries that are not meant to be run locally.
    The usefulness and functionality of the mocks has been demonstrated but additional work is required to fully incorporate them into the development workflow.

It is understood that the practice of storing notebooks, particularly the personal notebooks, will not scale into commissioning.
It is anticipated that this repo will split into multiple components such as example notebooks, operations-focused notebooks (where they will be run by operators to diagnose or characterize certain behaviour), and personal notebooks.
The details of this organization are beyond the scope of this technote.
Until the re-organisation is completed, tags will be made of the repo at least every 6 months or before/after major activities.
After each release, user will be asked to review and possibly remove notebooks older than 1 year to make sure stale notebooks are not lingering alongside the main working branch.


.. _Observing_Utilities:

Observing Utilities
====================

Observing utilities are user-defined methods that perform tasks that are not already part of the base control packages (the `Control Packages`_ section discusses this in further detail).
An example of functionality contained in a utility would be the reduction/analysis of an image.
In the use-case discussed in this document, the user defines methods that perform basic ISR on an image, finds the center of the star, and calculates the required offset.
In the cases where image reduction and/or analysis is required, specifically for ComCam and LSSTCamera images, the processing may utilize the `OCS Controlled Pipeline Service (OCPS) <https://dmtn-133.lsst.io/>`_, which is still undergoing design and development.
More details on it's use during development will be added once available.

The repo sanctioned for the development and use of such functions is the `ts_observing_utilities`_ repo, which follows an `LSST standard package format <https://github.com/lsst/templates>`_.
Users develop their functions on a branch and the functions shall go through a review (PR) process prior to being merged to the develop branch.
This area is designed to act as a staging area prior to having their functionality either moved into control packages, or promoted to sanctioned utilities which would be contained in the `ts_observatory_control`_ repo (discussed in the `Control Packages`_ section).

The development practices of this area are purposefully loose to promote rapid coding and integration.

.. Note::

    There is a `Python library <https://pypi.org/project/deprecation/>`_ available that allows developers and users to mark methods for deprecation using a decorator.
    It may be worth considering using this library to prevent bit-rot.


Required Testing
^^^^^^^^^^^^^^^^

Requirements on code prior to merging are minimal.
In short, the code should be runnable and should be documented at a level such that other people can identify what it does, as well as the inputs and outputs.


.. Important::

    Code in this repo is *not* allowed to be called by production level SAL Scripts *that are not on a ticket branch*.
    This is because changes in this repo do not require all tests in the production code areas to be run which could therefore lead to breakages.


.. _Control Packages:

Control Packages
================
Control Packages perform coordination of CSC functionality at a high-level.
An example of such an operation is slewing the telescope and dome, discussed in more detail below.
Because these packages (often written as classes) are used throughout many areas of operations, more significant levels of unit and integration testing are required; especially if utilities are contained outside the class.
High-level control packages live in their own repository (`ts_observatory_control`_).
These classes are written and tightly controlled by the T&S team.

As mentioned in the introduction, the master and develop branches of this codebase shall be entirely runnable at all times.

In the example use-case for this technote, the user wishes to take images with multiple instrument setups. '
Because the focus changes with different glass thicknesses and wavelength, this is the type of functionality that really should belong in the standard Control Package. However, while this use-case was being developed, that functionality didn't exist and was therefore developed in a utility (in `ts_observing_utilities`_).

To remedy this, the proper path forward is to request that the additional functionality be added.
To do this, the user should file a JIRA ticket with the requested functionality for review in the DM project with the team set to Telescope and Site.
Make sure to add interested parties as watchers/reviewers to ensure sufficient visibility.
This will trigger discussion on whether the functionality should indeed be implemented.
Upon conclusion of that discussion, a user can either wait for it to be implemented or make the changes themselves and submit a pull-request.

In the meantime, the utility in `ts_observing_utilities`_ shall remain until the functionality gets included in the Control Packages.
Once included, the utility should be deprecated and the appropriate code updated accordingly.

Control Package Examples
^^^^^^^^^^^^^^^^^^^^^^^^
The following are examples of classes written to perform basic control operations of the telescope, dome and instrument.

ATCS
-----
The `ATCS class <https://ts-observatory-control.lsst.io/py-api/lsst.ts.observatory.control.auxtel.ATCS.html?highlight=atcs>`_ contains methods that coordinate telescope and dome related CSCs. The class
includes methods that
capture complex activities in single lines of executable code such as slewing the telescope and dome (shown in the
example below), offsetting in multiple coordinate systems, starting/stopping of tracking etc.
Any required low-level (non-CSC) functionality should be pushed into these classes.

.. code-block:: python

    from lsst.ts.observatory.control.auxtel.atcs import ATCS
    atcs = ATCS()
    await atcs.start_task
    await atcs.slew_icrs(ra="20:25:38.85705", dec="-56:44:06.3230", sky_pos=0., target_name="Alf Pav")

Alternatively, the `ATCS` class also provides a `slew_object` method that queries
the object coordinate from `SIMBAD <http://simbad.u-strasbg.fr/simbad/>`_.

.. code-block:: python

    from lsst.ts.observatory.control.auxtel.atcs import ATCS
    atcs = ATCS()
    await atcs.start_task
    await atcs.slew_object(name="Alf Pav", sky_pos=0.)


LATISS
------
The `LATISS class <https://ts-observatory-control.lsst.io/py-api/lsst.ts.observatory.control.auxtel.LATISS.html>`_ coordinates the ATSpectrograph and ATCamera CSCs, taking various types of
images from a single command. This results in the proper metadata being published such that the image headers
are captured correctly.

.. code-block:: python

    from lsst.ts.observatory.control.auxtel.latiss import LATISS
    latiss = LATISS()
    await latiss.start_task
    exp_id = await latiss.take_engtest(exptime=10, filter='RG06', grating='empty_1')


.. _Control Utilities:

Control Package Utilities
^^^^^^^^^^^^^^^^^^^^^^^^^

Control Package Utilities are analogous to the utilities discussed in `Observing Utilities`_, but have been evolved and moved into the production code areas.
Sanctioned Control Utilities will exist at multiple levels.
These utilities will primarily be called by SAL Scripts for the scriptQueue, but not in all cases.
Top-level utilities will apply to both telescopes, all instruments, then each level down will have it's own utilities.
An example of this could (not necessarily will) be the centering utility described above, since the desired position for stars in LATISS will differ from the main telescope.

Utilities should be as atomic as possible and may not perform actions that get performed by the control classes (e.g. ATCS and LATISS), such as slewing the telescope.

The utilities live in the `ts_observatory_control`_ repo with the Control Classes.


Required Testing
----------------

All code in the `ts_observatory_control`_ requires documentation to a level where other developers can diagnose the utility and fix any issues that are resulting in failed tests.
This shall include a description of the utility, a description of the inputs/outputs, and depending on the complexity of the function, examples may be required.

Each utility shall come with a set of tests (and accompanying data if required), tests shall include:

- Validation of appropriate input types (dtypes)

    - Verification of appropriate input values are only required if the values are not checked/verified elsewhere (such
      as at lower levels (e.g. the CSCs).

- Testing of end-to-end functionality for the primary functions for appropriate inputs

    - E.g. does it correctly measure the centroid on a piece of test data to within a given tolerance?

- Testing that common edge cases are properly captured/treated

- Testing is *not* required for *all* possible input parameters and combinations


The following level of integration tests (on the NCSA-integration test stand) are also required:

- All SAL Scripts and utilities in the controls package shall successfully pass all tests.

    - Ideally this would be done automatically using a CI framework. If not available, then an artifact needs
      to be shown as part of PR
    - Tests have to pass **before merging** not just at the time of creating the PR.


.. TODO::
    DM is developing a way to do this and it will be explored if the solution is applicable here as well.
    For test data used in unit tests DM uses git-lfs to store repositories that are set up as eups packages.
    Another possible solution is Travis, which is used to test the LSST EFD helper class and/or Jenkins.
    Docker spins a temporary influxDB instance and loads test EFD data into it.
    A similar pattern could be loaded to test code that needs EFD data.


.. _Tasks:

SAL Scripts for the scriptQueue
===============================

The scriptQueue is the mechanism to run SAL Scripts in an automated fashion during commissioning and operations.
The level of robustness required for these SAL Scripts is divided among those still in development and those which are in full production.


SAL Scripts in Development
^^^^^^^^^^^^^^^^^^^^^^^^^^
SAL Scripts undergoing development live in the `ts_externalscripts`_ repo.
While in this repo, the SAL Scripts are permitted to call utilities in the `ts_observing_utilities`_ repository as it will often be the case that the user is developing utilities to be used with a SAL Script.
Of course, it may also call any of the functionality in the Control Package Repository (`ts_observatory_control`_).
Scripts and utilities in the `ts_externalscripts`_ and `ts_observing_utilities`_ areas are expected to follow a standard format/template and conform to proper standards (PEP8 and `TSSW Development Guide <https://tssw-developer.lsst.io/>`_ ).
Pushing from a ticket branch to the develop branch of the repo requires a review (PR).

Usage of the `ts_externalscripts`_ repository as opposed to a branch of `ts_standardscripts`_ is encouraged for when development is of higher complexity and will occur over a longer timespan.
For example, a SAL Script may be developed during a run but requires further development/testing in coming runs.
This was the case in developing the `latiss_cwfs_align` script for performing focus/collimation of the Auxiliary Telescope.
Because interfaces, CSCs and high-level classes were undergoing regular changes in this time, it was more practical to merge `latiss_cwfs_align` to the develop branch between runs and update it when applicable.


There will (probably) exist cases where a SAL Script will never be promoted to a production task.
In this case, the SAL Scripts shall be identified as such and will be subject to a higher level of documentation and required testing,
particularly against any possible utilities that may be deprecated.
Significant effort should be made to ensure that any persistent SAL Scripts in this repo do not require anything in the `ts_observing_utilities`_ repository as it will not be stable with time.

Required Testing
----------------

In order to merge a branch to the develop branch, each SAL Script shall:

- Have correctly populated metadata (e.g. author(s), semi-accurate run-times, description of goals, input parameters, output data etc.
- Have (and pass) a unit test demonstrating that is it is of proper format and capable of being executed

    - This is best accomplished using the BaseScriptTestCase helper class already available in `ts_standardscripts`_.
      This verifies the classes/functions conform with the baseclass and verifies the SAL Script won't fail due to syntax etc.
      It does not check format/readability/sensible inputs etc.

This can be accomplished by using the BaseScriptTestCase helper class already available in `ts_standardscripts`_.

No integration testing (on the NCSA-teststand) is strictly required, however, one would hope that the script has run successfully through the integration-test-stand or on the summit.


SAL Scripts in Production
^^^^^^^^^^^^^^^^^^^^^^^^^

SAL Scripts in full production are to be kept in the `ts_standardscripts`_ repository.
This is the last step in the development process.
SAL Scripts in this category are tightly controlled and standards are strictly enforced.
No production level SAL Script can call any utility in the `ts_observing_utilities`_ repository.
All called utilities shall be sanctioned Control Package Utilities.
All SAL Scripts in this repository shall be runnable at all times by any operator.
All code shall be documented at a level where other developers can diagnose the code and fix any issues that are resulting in failed tests.
This shall include a description of the SAL Script, a description of the inputs/outputs, and depending on the complexity of the function an example may be required.
All required metadata for the SAL Script shall be accurate (e.g. completion times).
The testing requirements discussed in the following section shall also be met.


Required Testing
----------------

In order to merge to develop the following level of testing shall be implemented and passing:

- Code shall be fully documented.

- Have (and pass) a unit test showing the SAL Script is of a format that is capable of being executed

    - This will use the helper class in already in `ts_standardscripts`_ (BasescriptTestCase).
      This verifies the classes/functions conform with the baseclass and verifies the SAL Script won't fail due to syntax etc.
      It does not check format/readability/sensible inputs

- Validation of inputs (checks dtypes not the values themselves)
- Unit testing of called utilities are not re-tested here, unless required by special circumstance


Integration tests (on the NCSA teststand):

- SAL Script shall run successfully through the integration-test-stand using a test dataset.

    - Standard usage modes of the SAL Script should have tests.
      Non-standard functionality tests not strictly required but strongly recommended.

- All other SAL Script and utilities shall also be successfully passing all unit tests and pass tests run on the test-stand.
  Tests have to pass **before merging** not just at the time of the pull-request.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
