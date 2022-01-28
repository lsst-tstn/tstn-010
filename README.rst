.. image:: https://img.shields.io/badge/tstn--010-lsst.io-brightgreen.svg
   :target: https://tstn-010.lsst.io
.. image:: https://github.com/lsst-tstn/tstn-010/workflows/CI/badge.svg
   :target: https://github.com/lsst-tstn/tstn-010/actions/
..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

#############################################
Observing Script and Method Development Cycle
#############################################

TSTN-010
========

This technote is to detail out the observing script and notebook development cycle from a simple test or idea, developed inside a notebook, to evolving into a method that could be called by other notebooks, and/or into a script to be called via the scriptQueue. Details include what repos should be used for development, and what level of testing/review is required to move to subsequent steps.

**Links:**

- Publication URL: https://tstn-010.lsst.io
- Alternative editions: https://tstn-010.lsst.io/v
- GitHub repository: https://github.com/lsst-tstn/tstn-010
- Build system: https://github.com/lsst-tstn/tstn-010/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally with `Sphinx`_:

.. code-block:: bash

   git clone https://github.com/lsst-tstn/tstn-010
   cd tstn-010
   pip install -r requirements.txt
   make html

.. note::

   In a Conda_ environment, ``pip install -r requirements.txt`` doesn't work as expected.
   Instead, ``pip`` install the packages listed in ``requirements.txt`` individually.

The built technote is located at ``_build/html/index.html``.

Editing this technical note
===========================

You can edit the ``index.rst`` file, which is a reStructuredText document.
The `DM reStructuredText Style Guide`_ is a good resource for how we write reStructuredText.

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published technote at https://tstn-010.lsst.io will be automatically rebuilt whenever you push your changes to the ``main`` branch on `GitHub <https://github.com/lsst-tstn/tstn-010>`_.

Updating metadata
=================

This technote's metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the technote's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

Using the bibliographies
========================

The bibliography files in ``lsstbib/`` are copies from `lsst-texmf`_.
You can update them to the current `lsst-texmf`_ versions with::

   make refresh-bib

Add new bibliography items to the ``local.bib`` file in the root directory (and later add them to `lsst-texmf`_).

.. _Sphinx: http://sphinx-doc.org
.. _DM reStructuredText Style Guide: https://developer.lsst.io/restructuredtext/style.html
.. _this repo: ./index.rst
.. _Conda: http://conda.pydata.org/docs/
.. _lsst-texmf: https://lsst-texmf.lsst.io
