..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Recently, in DMTN-085, the QA Strategy Working Group (QAWG) made specific recommendations to improve the SQuaSH metrics dashboard. This technote presents an overview of the current implementation and a plan to implement what is missing.

Introduction
============

.. qawg-rec::

Summary of the QAWG recommendations to SQuaSH
=============================================

.. _qawg-rec-34:

QAWG-REC-34
    | SQuaSH should issue alerts to developers and key stakeholders on regressions in important metric values.

.. _qawg-rec-35:

QAWG-REC-35
    | Provide a single, reliable source of documentation describing the SQuaSH system and a vision for its use in DM-wide metric tracking

.. _qawg-rec-36:

QAWG-REC-36
    | The SQuaSH system should be closely coupled to the drill-down environment; in particular, the former should use the latter to enable drill-down functionality into particular metric values.

.. _qawg-rec-37:

QAWG-REC-37
    | It must be possible to submit metrics to SQuaSH from arbitrary pipeline execution environment.

.. _qawg-rec-38:

QAWG-REC-38
    | SQuaSH should be able to store and display appropriate metric values per DataId.


These recommendations help to define the scope and technical requirements for SQuaSH. In particular, |37| and  |38| have implications on how metric values are stored and visualized, including metadata from the verification jobs and the execution environment. |34| requires an automated regression detection and a notification system.  |36| suggests a new mechanism to expose metrics stored in SQuaSH to the notebook aspect of the LSST Science Platform (LSP) for exploratory analysis. Finally, |35| requires better user documentation.

Another general recommendation made by the QAWG is that the drill-down capability should be removed from SQuaSH and implemented in external dashboards or the notebook aspect of the LSP.




.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References


.. |34| replace:: :ref:`QAWG-REC-34 <qawg-rec-34>`
.. |35| replace:: :ref:`QAWG-REC-35 <qawg-rec-35>`
.. |36| replace:: :ref:`QAWG-REC-36 <qawg-rec-36>`
.. |37| replace:: :ref:`QAWG-REC-37 <qawg-rec-37>`
.. |38| replace:: :ref:`QAWG-REC-38 <qawg-rec-38>`
.. _SQR-009: https://sqr-009.lsst.io



.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
