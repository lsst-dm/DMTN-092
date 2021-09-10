..
  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

:tocdepth: 1

.. sectnum::

.. _ap-interfaces-intro:

Introduction
============

This document describes how the Alert Production or Raw Calibration Validation pipelines are invoked and how they communicate with the Butler and Prompt Products Database.
It also describes how metrics they produce are returned to Summit systems.


.. _invoking-ap:

Invoking the Alert Production Pipeline
======================================

This section describes the interface between the Prompt Processing system and the Alert Production pipeline that allows the former to invoke the latter.

.. _invoking-ap-batch:

Initial batch implementation
----------------------------

Until a full-featured Prompt Processing execution framework is available, the Alert Production pipeline will be invoked as batch jobs submitted to HTCondor or SLURM.
One batch job will be submitted per camera CCD (9 for ComCam, 189 for LSSTCam, or 193 if guide CCDs are used for science).
The batch job will execute a shell script which is in turn expected to execute one or more ``pipetask`` commands.
The script must accept as arguments the name of the CCD to be processed, the RA and declination of the telescope boresight, the name of the filter to be used for the observation, if any, and the anticipated exposure time.
The script will also accept as parameters the location of the Butler responsible for providing the input image or images for the Alert Production pipeline and the name(s) of the image(s) to be retrieved.
The values of all of those arguments will be provided by the Prompt Processing system based on the ``nextVisit`` event.
All other information about the image (including detailed information about the shutter motion used to compute more accurate exposure times) must be obtained from the image headers.

The batch job will be submitted no later than the ``startIntegration`` event, although there may be some latency until it starts running.

.. _invoking-ap-dynamic:

Later dynamic implementation
----------------------------

When a Prompt Processing execution framework is available, it will directly invoke the command given above, without the intermediation of a batch processing system.
One command will be executed per camera CCD.
All of the same arguments will be provided.

The command should be executed at ``nextVisit`` time, although it's possible that for technical reasons it will have to be delayed until ``startIntegration``.

.. _invoking-raw-calib:

Raw calibration pipeline
------------------------

The raw calibration pipeline will be invoked with the "intent" (dark, flat, CBP, etc.) of the image, the name of the CCD, the name of any filter in place, the anticipated exposure time, the location of the Butler, and the name of the image.
All other information about the image (including detailed information about the shutter motion used to compute more accurate exposure times) must be obtained from the image headers.

The command should be executed at ``startIntegration`` time; there are no ``nextVisit`` events for raw calibration images.

.. _retrieving-ap-template-images:

Retrieving Template and Calibration Images
==========================================

The Alert Production pipeline is expected to retrieve template and master calibration images via a configured Data Butler.
A special Datastore is anticipated to be provided for these images that would have higher performance and greater uptime than the normal Data Backbone.

.. _retrieving-ap-science-images:

Retrieving Science Images
=========================

The Alert Production pipeline retrieves science images (one or two per visit) via the Data Butler provided as an argument.
The `get()` call to that Data Butler must block until the image is received and registered.
The Alert Production should timeout and exit if no image has been received within a configurable period, e.g. 2 minutes.

.. _retrieving-ppdb-items:

Retrieving DIASources, DIAObject, and DRP Objects
=================================================

The Alert Production pipeline should initialize the PPDB interface (a Python module) when it is invoked, using the boresight location and CCD name to provide a spatial region to be processed.

Historic DIASources, DIAObjects, and the identifiers for DRP Objects should then be retrieved immediately prior to their usage in the association portion of the AP pipeline, using the PPDB interface.

New DIASources and versions of DIAObjects should then be written, again using the PPDB interface.
The values/columns for these objects need to be in the form specified by the DPDD (LSE-163), with appropriate units and descriptions.

The PPDB interface is currently specified to use ``pandas``.

.. _writing-alerts:

Writing Alerts to Alert Distribution
====================================

The Alert Production pipeline should convert DIASources and associated history and postage stamp images into Alerts in Apache Avro format.
It should then issue Kafka messages to convey them to Alert Distribution and its downstream filters and brokers.

.. _dispatching-metrics:

Dispatching Metrics to the EFD
==============================

The metrics required by :lse:`72` §2.1.1 will be written out by the pipelines as small Butler datasets in JSON format.
The ``faro`` and ``lsst.verify`` packages are the reference for how to do this.
The resulting metrics will be dispatched to the Engineering and Facilities Database by communicating directly with its Summit InfluxDB instance.
Code similar to that in ``lsst.verify`` that dispatches to SQuaSH will be used.
It will be necessary to ensure that appropriate information from the Butler data ID is made available to associate the metric with a visit or an exposure and/or a detector.

Sending these metrics as SAL messages is not thought to be necessary, as CSCs requiring the metric information are likely to want their time history (best obtained by an InfluxDB query) and can request single values if needed..
This simplification also eliminates the translations from pipeline metric to SAL message, from SAL message to Kafka, and from Kafka to InfluxDB.

Replication of these metrics to other EFD instances, including the relational Transformed EFD (:sqr:`58`), will still occur.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
