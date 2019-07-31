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


Summary of the QAWG recommendations to SQuaSH
=============================================

Recently, in :dmtn:`085` :cite:`DMTN-085`, the QA Strategy Working Group (QAWG) made specific recommendations to improve the SQuaSH metrics dashboard.


.. _qawg-rec-34:

QAWG-REC-34
    | SQuaSH should issue alerts to developers and key stakeholders on regressions in important metric values.

.. _qawg-rec-35:

QAWG-REC-35
    | Provide a single, reliable source of documentation describing the SQuaSH system and a vision for its use in DM-wide metric tracking.

.. _qawg-rec-36:

QAWG-REC-36
    | The SQuaSH system should be closely coupled to the drill-down environment; in particular, the former should use the latter to enable drill-down functionality into particular metric values.

.. _qawg-rec-37:

QAWG-REC-37
    | It must be possible to submit metrics to SQuaSH from arbitrary pipeline execution environments.

.. _qawg-rec-38:

QAWG-REC-38
    | SQuaSH should be able to store and display appropriate metric values per DataId.


These recommendations help to define the scope and technical requirements for SQuaSH. In particular, |37| and |38| have implications on how metric values are stored and visualized, including metadata from the verification jobs and the execution environment. |34| requires an automated regression detection and a notification system. |36| suggests a workflow that starts in SQuaSH to find a misbehaving metric and then spawns an instance of the drill-down environment with enough information to look into why that particular metric is an excursion. Finally, |35| requires better user documentation.

Another general recommendation made by the QAWG is that the drill-down capability should be removed from SQuaSH and implemented in external dashboards or the notebook aspect of the LSST Science Platform (LSP).


Adopting the InfluxData stack
=============================

In DM-16223_, we investigate open source solutions that could be adopted to achieve the functionalities required in SQuaSH. Among them, the InfluxData_ stack presents the most exciting monitoring capabilities. This section describes the status of adopting the InfluxData stack in SQuaSH, in particular, InfluxDB_, Chronograf_, and Kapacitor_ components. We describe what has changed in SQuaSH so far, and propose a roadmap to implement what is pending.


.. _influx-db:

InfluxDB, a time-series database
--------------------------------

InfluxDB_ is designed to store time-series efficiently. The realization that metric values and metadata indexed by time is time-series data makes a time-series database the natural choice for SQuaSH.

:sqr:`009` :cite:`SQR-009` describes how we `map metric values and metadata to InfluxDB concepts <https://sqr-009.lsst.io/#storing-results-in-squash>`_ like measurements, fields, and tags. In particular, arbitrary metadata added to verification jobs are mapped to InfluxDB tags and then we can use those tags in the InlfuxQL queries to distinguish metric values measured on different environments, pipelines, datasets, etc.

.. todo:: DM-16315_ Document the list of metadata tags used within SQuaSH.

This implementation has been tested with Alert Production (AP) and Data Release Production (DRP) metrics. Following |38|, we use "metadata tags" to annotate the DataId in which metric values are computed (e.g., ``ccdnum``, ``visit``) and then we can filter or aggregate metric values across DataId's.

Example of an InfluxQL_ query to retrieve metric values per ``ccdnum``:

.. code-block:: SQL

  SELECT "totalUnassociatedDiaObjects"
  FROM "squash"."autogen"."ap_association"
  WHERE  ("ccdnum"='10' OR "ccdnum"='5' OR "ccdnum"='56')
  GROUP BY "ccdnum"

Example of an InfluxQL_ query to aggregate metric values across multiple ``ccdnum``'s:

.. code-block:: SQL

  SELECT mean("totalUnassociatedDiaObjects")
  FROM "squash"."autogen"."ap_association"
  WHERE  ("ccdnum"='10' OR "ccdnum"='5' OR "ccdnum"='56')
  GROUP BY time(1d)

The aggregation example uses the ``mean()`` InfluxQL_ function to aggregate the metric values for the ``ccdnum``'s in the ``WHERE`` clause, and does that in time intervals of ``1d``, which is the cadence we get metric values from CI.

.. note::

  The timestamp used to write metric values to InfluxDB has implications to the aggregation. In DM-17767_, we use the CI pipeline run time as the InfluxDB timestamp. That ensures we write all metric values from a given CI run with the same timestamp in InfluxDB.

DM-16775_ implements a notebook that exercises the mapping described in :sqr:`009` :cite:`SQR-009`. There's a pending ticket (DM-19605_) to improve the mapping of metric names to InfluxDB fields, which greatly simplifies the InfluxQL queries.

Despite the adoption of InfluxDB, the SQuaSH API specification remains unchanged, and so the clients that use it. The main addition is the code that formats the data and writes to the corresponding InfluxDB instance.

To complete this work we need to implement DM-18060_ to recreate the SQuaSH production database using the mapping described in :sqr:`009` :cite:`SQR-009`, and re-ingest the verification existing jobs in the current SQuaSH database.

.. todo:: Deploy a separate InfluxDB instance for each SQuaSH instance (dev, test, prod).

In addition to InfluxDB, SQuaSH has a `MySQL database`_  that now figures like a `context database` storing metric definitions and specifications in addition to verification job and environment metadata.

InfluxDB also provides an HTTP API. The InfluxDB HTTP API can be used directly in the notebook aspect of the LSP for querying metric data. We are also considering other data access mechanisms like the Butler and the DAX APIs (see also :ref:`metric-data-access`)

.. note::
  Currently, we write metric values and metadata in both the MySQL and InfluxDB database instances. We can either drop the ``measurements`` table in the `MySQL database`_ or decide to use it to expose the results through the `IVOA Table Access Protocol <http://www.ivoa.net/documents/TAP/>`_.


From the recommendation that we should not implement drill-down capabilities in SQuaSH, we could also drop the support for data blobs, unless we still need that to store other artifacts produced by the verification packages.

.. todo:: Define and create a ticket to drop the support for data blobs in SQuaSH.


Chrognograf, a replacement for the SQuaSH frontend
--------------------------------------------------

Chronograf_ is the graphical user interface (GUI) for the InfluxData_ stack. The `Explore tool`, in particular, has proven to be intuitive and straightforward to query AP and DRP metrics. These queries can be saved and organized in dashboards (e.g., DM-16942_). Chronograf also provides an intuitive interface to Kapacitor_ for creating alerting rules and notifications.

Customizations in the Chronograf interface for SQuaSH so far include the support to markdown content in table cells (DM-18343_) and thus the ability to display `code changes` in the new interface (DM-18525_) as in the original Bokeh_-based implementation.

.. todo:: Redirect http://squash.lsst.codes to the Chronograf interface for SQuaSH.

.. todo:: Deploy a separate InfluxDB instance for each SQuaSH instance (dev, test, prod).

For the moment, Chronograf did not present any significant limitations for displaying metrics. We still need to implement DM-18594_ to display specification thresholds in Chronograf.

However, we might consider alternatives like Grafana_ for creating dashboards, which is straightforward to implement as Grafana includes a data source for InfluxDB. Either Chronograf or Grafana seems like a good option for replacing the original SQuaSH frontend saving several hours of development time for the project.

Kapacitor, metric regression and notification system
----------------------------------------------------

Kapacitor_ is an open-source data processing framework that makes it easy to detect regressions on metric values and send notifications.

Kapacitor uses a language called TICKscript_ to define tasks. Tasks can run on streaming data (e.g., as metric values are written to InfluxDB) or as batch jobs on data stored in InfluxDB.

An exciting feature of Kapacitor is the `record/replay capability <https://docs.influxdata.com/kapacitor/v1.5/working/cli_client/#data-sampling>`_ to test the tasks before enabling them. This feature is useful to make sure the tasks work as expected, and the notification messages are well-formed.

A task typically defines the data to test through an InfluxQL_ query. The possible tests are:

  - **Threshold** -- when the returned value is compared to a reference value.
  - **Relative** -- when the returned value change by an absolute or relative amount compared with a previous value.
  - **Deadman** -- send notification if data is missing for a certain amount of time.

Chronograf presents an intuitive, however incomplete, interface to create and manage tasks (a.k.a alert rules). Kapacitor itself, on the other hand, provides a complete `HTTP API <https://docs.influxdata.com/kapacitor/v1.5/working/api/>`_  to manage tasks.

In DM-16293_, we investigate how to use the Kapacitor HTTP API to create tasks programmatically using the metric specifications from the SQuaSH API.

Following is an example of a streaming task to test ``ap_association.AssociationTime`` metric values. The task triggers a notification when the metric value is larger than the specified threshold. In this example, the notification is sent to the ``#dm-squash-alerts`` slack channel.

.. code-block:: javascript

  var name = 'Association time alert'
  var db = 'squash-prod'
  var rp = 'autogen'
  var measurement = 'ap_association'
  var groupBy = ['visit', 'ccdnum', 'ci_dataset']
  var whereFilter = lambda: TRUE

  var message = '{{.Name}} is {{.Level}} on build #{{ index .Tags "ci_id" }}: AssociationTime = {{ index .Fields "value" | printf "%0.2f s" }} for {{.Group}}'

  var triggerType = 'threshold'
  var crit = 5

  var data = stream
      |from()
          .database(db)
          .retentionPolicy(rp)
          .measurement(measurement)
          .groupBy(groupBy)
          .where(whereFilter)
      |eval(lambda: "ap_association.AssociationTime")
          .as('value')
  var trigger = data
      |alert()
          .crit(lambda: "value" > crit)
          .message(message)
          .stateChangesOnly()
          .slack()
          .channel('#dm-squash-alerts')

Example of a notification message produced by this task:

    *ap_association is CRITICAL on build #279:
    AssociationTime = 5.42s for ccdnum=56, ci_dataset=CI-HiTS2015, visit=411371*


|34| suggests a “subscription list” for each metric to be defined, and the key stakeholders automatically be added to it for all metrics deriving directly from high-level requirements. This could be achieved by sending notifications to specific slack channels for example, notification about regression on AP metrics are sent to ``#dm-alert-prod``, notifications about regression on DRP metrics to ``#dm-drp``, etc.


Supporting multiple execution environments
==========================================

To be generally useful for the verification activities, SQuaSH must support multiple execution environments.

The following project environments are currently supported:

* Jenkins CI
* LDF

SQuaSH captures environment variables from these environments and use them as metadata associated with the metric values.

.. todo:: Document the required environment variables in each situation and the corresponding metadata tags used by SQuaSH.

SQuaSH has the concept of runs. A run may contain results from several verification jobs executed on a given environment. For example a ``GET`` request to ``/jenkins/<run_id>`` or to ``/lfd/<run_id>`` will retrieve all the verification jobs in that run.

In DM-18505_, we add support for a local execution environment. That allows DM developers to run verification jobs in the notebook aspect of the LSP or from their laptop and dispatch results to SQuaSH. We plan to implement one execution environment per user to have have in SQuaSH auto-incremental run IDs per user. The user name registered in SQuaSH is going to be added as metadata to the environment. This implementation fulfills |37|.

.. note::

  Dispatching results to SQuaSH requires auth access to the SQuaSH API. Currently, the only mechanism to register new users is interacting to the SQuaSH API. That can be simplified using the GitHub OAuth2 provider in SQuaSH .

.. note::

  Because the local execution environment is not a controlled environment like the Jenkins CI or the LDF, we can not capture information such as code version or configuration.

.. _metric-data-access:

Connecting SQuaSH to the drill-down environment
===============================================

:dmtn:`085` :cite:`DMTN-085` describes drill-down workflows to debug processing problems and investigate the effects of new algorithms. It recommends the implementation of a "browser-based interactive dashboard that can run on any pipeline output data repository (or comparison of two repositories) to quickly diagnose the quality of the data processing". Here, the drill-down system is referred to merely as QA dashboard.

.. note::

  The QA dashboard is based on the PyViz_ and HoloViz_ ecosystem and is designed in such a way that it is easy to run the interactive visualization tools either as standalone dashboards or from a notebook.

To connect SQuaSH and the QA dashboard in a meaningful way, they need to share a subset of metrics.  Those metrics must have the same definition, must be computed by the same code, configuration, and run on the same data. Finally, metric values must be tested against the same specifications so that both systems indicate consistent metric regressions.

In other words, to accomplish the above we need to combine information from the Workflow System and the Verification Framework. The Workflow System runs the Pipeline Tasks on a controlled environment and associates code version and configuration with a run ID. The Verification Framework defines metrics and specifications and persists the metric values in verification jobs in the output data repository of that run. The verification jobs are submitted to SQuaSH along with the run ID and are used by the QA dashboard when analyzing the output data repository.

This way, the run ID links the SQuaSH and the QA dashboard. We also assume a service provided by the Workflow System in which the QA dashboard can introspect the path to the output data repository, code version, and configuration from the run ID.

.. note::

  Such a service is generaly useful. To mention another use case, we envision using the notebook-based report system :sqr:`023` :cite:`SQR-023` for generating periodic reports (e.g., :sqr:`026` :cite:`SQR-026`) where the run ID is a template variable.

SQuaSH's primary job is then to discover run IDs that present misbehaving metrics. Given the run ID, the  QA dashboard can get the information needed from the Workflow System for further investigation.

Given the assumptions above, to fulfill |36|, the minimum set of information that SQuaSH needs to store is:

  * Run ID;
  * Name of the execution environment;
  * DataId's associated with metric values.

The Run ID uniquely identifies the output data repository, the code version and the configuration run for a given environment. The name of the environment is used to distinguish runs executed on controlled environments (e.g., LDF) from runs performed on user local environments. Finally, DataId's associated with metric values make it possible to spawn an instance of the QA dashboard on a particular DataId.

From the point of view of SQuaSH, |36| is easy to implement, in fact for the Jenkins CI environment we already store the run ID, the name of the environment and DataIds as metadata tags associated with metric values (see :ref:`influx-db`).

The InfluxDB HTTP API is the recommended API to query metric values in SQuaSH, and the SQuaSH REST API is the recommended API to query metrics definition and specifications. Currently this is done by interacting directly with the APIs using the Python Requests_ module. This situation can be improved by implementing a SQuaSH Python client as a more user friendly interface to retrieve metric data.


SQuaSH documentation
====================

|35| recommends a single, reliable source of documentation describing the SQuaSH system and a vision for its use in DM-wide metric tracking.

We understand |35| requests SQuaSH documentation for DM developers. We plan to implement user documentation at https://squash.lsst.io. In addition, we need to complete the SQuaSH design documentation :sqr:`009` :cite:`SQR-009` and add deployment instructions to that as well.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

References
==========

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa


.. _InfluxData: https://www.influxdata.com/
.. _InfluxDB: https://docs.influxdata.com/influxdb/v1.7/
.. _InfluxQL: https://docs.influxdata.com/influxdb/v1.7/query_language/
.. _Grafana: https://grafana.com/docs/
.. _Chronograf: https://docs.influxdata.com/chronograf/v1.7/
.. _Kapacitor: https://docs.influxdata.com/kapacitor/v1.5/
.. _TICKScript: https://docs.influxdata.com/kapacitor/v1.5/tick/introduction/
.. _MySQL database: https://sqr-009.lsst.io/#the-squash-context-database/
.. _Bokeh: https://bokeh.pydata.org/en/latest/
.. _PyViz: https://pyviz.org/
.. _HoloViz: https://holoviz.org/
.. _Requests: https://2.python-requests.org/en/master/

.. _DM-16223: https://jira.lsstcorp.org/browse/DM-16223/
.. _DM-17767: https://jira.lsstcorp.org/browse/DM-17767/
.. _DM-16775: https://jira.lsstcorp.org/browse/DM-16775/
.. _DM-19605: https://jira.lsstcorp.org/browse/DM-19605/
.. _DM-18060: https://jira.lsstcorp.org/browse/DM-18060/
.. _DM-16942: https://jira.lsstcorp.org/browse/DM-16942/
.. _DM-18343: https://jira.lsstcorp.org/browse/DM-18343/
.. _DM-18525: https://jira.lsstcorp.org/browse/DM-18525/
.. _DM-16315: https://jira.lsstcorp.org/browse/DM-16315/
.. _DM-18505: https://jira.lsstcorp.org/browse/DM-18505/
.. _DM-16293: https://jira.lsstcorp.org/browse/DM-16293/

.. |34| replace:: :ref:`QAWG-REC-34 <qawg-rec-34>`
.. |35| replace:: :ref:`QAWG-REC-35 <qawg-rec-35>`
.. |36| replace:: :ref:`QAWG-REC-36 <qawg-rec-36>`
.. |37| replace:: :ref:`QAWG-REC-37 <qawg-rec-37>`
.. |38| replace:: :ref:`QAWG-REC-38 <qawg-rec-38>`

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
