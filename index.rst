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

Recently, in :dmtn:`085` :cite:`DMTN-085`, the QA Strategy Working Group (QAWG) made specific recommendations to improve the SQuaSH metrics dashboard.

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


These recommendations help to define the scope and technical requirements for SQuaSH. In particular, |37| and |38| have implications on how metric values are stored and visualized, including metadata from the verification jobs and the execution environment. |34| requires an automated regression detection and a notification system.  |36| suggests a new mechanism to expose metrics stored in SQuaSH to the notebook aspect of the LSST Science Platform (LSP) for exploratory analysis. Finally, |35| requires better user documentation.

Another general recommendation made by the QAWG is that the drill-down capability should be removed from SQuaSH and implemented in external dashboards or the notebook aspect of the LSP.


Adopting the InfluxData stack
=============================

In DM-16223_, we investigate open source solutions that could be adopted to achieve the functionalities required in SQuaSH. Among them, the InfluxData_ stack presents the most exciting monitoring capabilities. This section describes the status of adopting the InfluxData stack in SQuaSH, in particular, InfluxDB_, Chronograf_, and Kapacitor_. We describe what has changed in SQuaSH so far, and actions that are still pending.

InfluxDB, a time-series database
--------------------------------

InfluxDB_ is designed to store time-series efficiently. The realization that metric values and metadata indexed by time is time-series data, makes a time-series database the natural choice for SQuaSH.

:sqr:`009` :cite:`SQR-009` describes how we `map metric values and metadata to InfluxDB concepts <https://sqr-009.lsst.io/#storing-results-in-squash>`_ like measurements, fields, and tags. In particular, we store arbitrary metadata in the verification job as InfluxDB tags (e.g., ``pipeline``, ``dataset``, ``filter``, ``ccdnum``, and ``visit``) and then we can use these tags to filter and group metric values.

..todo:: DM-16315_ Details on how we do the mapping are not relevant for the user and should be removed from the user documentation, what is important is the list of metadata used to annotate the verification jobs.

This implementation has been tested with Alert Production (AP) and Data Release Production (DRP) metrics and completely satisfies |38|.

Example of a query to retrieve metric values per ``ccdnum``:

.. code-block:: SQL

  SELECT "totalUnassociatedDiaObjects"
  FROM "squash"."autogen"."ap_association"
  WHERE  ("ccdnum"='10' OR "ccdnum"='5' OR "ccdnum"='56')
  GROUP BY "ccdnum"

Example of a query to aggregate metric values across multiple ``ccdnum``'s:

.. code-block:: SQL

  SELECT mean("totalUnassociatedDiaObjects")
  FROM "squash"."autogen"."ap_association"
  WHERE  ("ccdnum"='10' OR "ccdnum"='5' OR "ccdnum"='56')
  GROUP BY time(1d)

The aggregation example uses the ``mean()`` `InfluxQL function`_  to aggregate the metric values for the ``ccdnum``'s in the ``WHERE`` clause, and does that in time intervals of ``1d``, which is the cadence we get metric values from CI. Note that the timestamp you use to write metric values to InfluxDB has implications for the aggregation. In DM-17767_, we use the CI pipeline run time as the InfluxDB timestamp. That ensures we write all metric values with the same timestamp in InfluxDB.

DM-16775_ implements a notebook to exercise the mapping described in :sqr:`009` :cite:`SQR-009`. There's a pending ticket DM-19605_ to implement the mapping of metric name to InfluxDB fields that simplifies the InfluxQL queries.

Despite adopting InfluxDB, the SQuaSH API specification remains unchanged, and so the clients that use the SQuaSH API. The main addition is the code that formats the data to the InlfuxDB line protocol and writes to the corresponding InfluxDB instance.

To complete this work we need to implement DM-18060_ to recreate the SQuaSH production database to use the mapping described in :sqr:`009` :cite:`SQR-009`, and re-ingest the verification existing jobs in the current SQuaSH database.

.. todo:: Deploy a separate InfluxDB instance for each SQuaSH instance (dev, test, prod).

In addition to InfluxDB, SQuaSH has a `MySQL database`_  that is now used more like a `context database` storing metric definitions and specifications in addition to job and execution and environment metadata.

InfluxDB already provides an HTTP API and an `SQL-like query language`_  to access the data. The InfluxDB HTTP API can be used directly in the notebook aspect of the LSP for querying Science Pipeline metrics. We are also considering other data access mechanisms like the Butler and the DAX APIs.

.. note::
  Currently, we write metric values and metadata in both the MySQL and InfluxDB database instances. We can either drop the ``measurements`` table in the `MySQL database`_ or decide to use this database to expose the results through TAP.

.. todo:: Design of metric data access from the LSP.

From the recommendation that we should not implement drill-down capabilities in SQuaSH, we can safely drop the support for data blobs from SQuaSH.

.. todo:: Create ticket to drop the support for data blobs in SQuaSH.


Chrognograf, a replacement for the SQuaSH frontend
--------------------------------------------------

Chronograf_ is the interface for the InfluxData_ stack. The `Explore tool`, in particular, has proven to be intuitive and straightforward to query AP and DRP metrics. These queries can be saved and organized in dashboards (e.g., DM-16942_). Chronograf also provides an intuitive interface to Kapacitor_ for creating alerting rules and notifications.

Customizations in the Chronograf interface for SQuaSH include the support to markdown content in table cells (DM-18343_) and thus the ability to display `code changes` in the new interface (DM-18525_) as in the Bokeh_-based SQuaSH implementation.

.. todo:: Redirect http://squash.lsst.codes to the Chronograf interface for SQuaSH.

.. todo:: Deploy a separate InfluxDB instance for each SQuaSH instance (dev, test, prod).

For the moment, Chronograf did not present any significant limitations for displaying metrics.

.. todo:: Display of specification thresholds in Chronograf (DM-18594)

However, we might consider alternatives like Grafana_ for creating dashboards, which is straightforward to implement as Grafana includes a data source for InfluxDB. Either Chronograf or Grafana seems like a good option for replacing the original SQuaSH frontend saving several hours of development time for the project.

Kapacitor, metric regression and notifications
----------------------------------------------


Supporting multiple execution environments
==========================================

To be generally useful for the verification activities, SQuaSH must support multiple execution environments.

The following project environments are currently supported:

* Jenkins CI
* LDF

SQuaSH captures environment variables from these environments and use them as metadata associated with the metric values.

.. todo:: Document the required environment variables in each situation and the corresponding metadata tags used by SQuaSH.

SQuaSH has the concept of runs. A run may contain results from several verification jobs executed on a given environment. For example a ``GET`` request to ``/jenkins/<run_id>`` or to ``/lfd/<run_id>`` will retrieve all the verification jobs in that run.

Adding support to local execution environment allows DM developers to run verification jobs in the notebook aspect of the LSP or from their laptop and submit the results to SQuaSH. Because the local execution environment is not a controlled environment like the Jenkins CI or the LDF, we can not capture information like code version.

In DM-18505_, we add support for a local execution environment and this implementation fulfills |37|.









.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

References
==========

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa


.. _InfluxData: https://www.influxdata.com/
.. _InfluxDB: https://www.influxdata.com/time-series-platform/
.. _InluxQL function: https://docs.influxdata.com/influxdb/v1.7/query_language/functions/
.. _Chronograf: https://www.influxdata.com/time-series-platform/chronograf/
.. _Kapacitor: https://www.influxdata.com/time-series-platform/kapacitor/
.. _MySQL database: https://sqr-009.lsst.io/#the-squash-context-database/
.. _SQL-like query language: https://docs.influxdata.com/influxdb/v1.7/query_language/
.. _Bokeh: https://bokeh.pydata.org/en/latest/

.. _DM-16223: https://jira.lsstcorp.org/browse/DM-16223/
.. _DM-17767: https://jira.lsstcorp.org/browse/DM-17767/
.. _DM-16775: https://jira.lsstcorp.org/browse/DM-16775/
.. _DM-19605: https://jira.lsstcorp.org/browse/DM-19605/
.. _DM-18060: https://jira.lsstcorp.org/browse/DM-18060/
.. _DM-16942: https://jira.lsstcorp.org/browse/DM-16942/
.. _DM-18343: https://jira.lsstcorp.org/browse/DM-18343/
.. _DM-18525: https://jira.lsstcorp.org/browse/DM-18525/
.. _DM-16315: https://jira.lsstcorp.org/browse/DM-16315/

.. |34| replace:: :ref:`QAWG-REC-34 <qawg-rec-34>`
.. |35| replace:: :ref:`QAWG-REC-35 <qawg-rec-35>`
.. |36| replace:: :ref:`QAWG-REC-36 <qawg-rec-36>`
.. |37| replace:: :ref:`QAWG-REC-37 <qawg-rec-37>`
.. |38| replace:: :ref:`QAWG-REC-38 <qawg-rec-38>`

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
