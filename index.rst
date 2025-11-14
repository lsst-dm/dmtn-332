############################################
Components of LSST image and catalog row IDs
############################################

.. abstract::

   This technotes describes how integer IDs for detector-level images and catalog rows (Sources and Objects) are built in the LSST pipelines from a combination of dates, camera sequence numbers, release IDs, detection counters, and other components.

.. _observation-ids:

Observation IDs
===============

Rubin Observatory has two concepts for a logical observation: an ``exposure`` is a low-level concept that can refer to any observation (including raw calibrations), while a ``visit`` is a set of one or more *on-sky* ``exposure`` "snaps" with *exactly* the same pointing, such that the images could be combined via a simple sum, as if they were taken in a single longer ``exposure``.
In the vast majority of observations taken to date (and in fact all observations taken with LSSTCam), there is only one ``exposure`` for each ``visit``.
While ``raw`` images are identified by their ``exposure``, almost all processed images are identified by a ``visit``, and hence the ``visit`` ID is the one science users will typically interact with.

The integer IDs for ``visits`` and ``exposures`` are formed in exactly the same way, so that for a ``visit`` with a single ``exposure`` their IDs are identical: an 11-digit integer that is the decimal-digit concatentation:

- ``day_obs``: a ``YYYYMMDD`` date that remains constraint throughout observatory night, itself decimal-digit concatenated into an integer;
- ``seq_num``: a zero-padded integer counter that resets at the beginning of each ``day_obs``.

For example, ``2025042900332`` is the integer ID of a single-snap ``exposure``/``visit`` taken the night of April 29, 2025, with ``seq_num=332``.

In the rare case where we observed multiple ``exposures`` with the intent that we would treat them as a single ``visit``, but later decided to reinterpret each ``exposure`` as its own ``visit``, the ``visit`` that holds the first ``exposure`` in the sequence *only* has its ID prefixed with a ``9``.
In all other cases the ``visit`` ID is the ID of the first ``exposure`` in the ``visit``.

Observations taken with the non-default Camera Control System (CCS; engineering data only) or variuos low-level simulators are identified by artificially incrementing the millenium in the ``day_obs`` (i.e. ``day_obs=30240101`` is the CCS ``day_obs`` value for observations taken the night of January 1, 2024).

This scheme applies to LSSTCam, LSSTComCam (the single-raft commissioning camera that was on the sky in fall of 2024), and LATISS (the Rubin Observatory auxillary telescope instrument).

.. _packed-detector-observation-ids:

Packed Detector-Observation IDs
===============================

In most contexts where we need to identify a ``{visit, detector}`` or ``{exposure, detector}`` combination, we prefer to use a tuple over a single integer for readability and clarity.
But in some contexts an integer is necessary, and for these we opt to repack various terms in the ``day_obs`` and ``seq_num`` more tightly, as the most common use case is to additionally pack a per-detector ``Source`` counter (see :ref:`source-and-object-ids`) into the same 64-bit integer, and bits are at a premium.
A packed detector-observation identifier is computed by the following pseudocode::

   packed = (
      detector + n_detectors * (
         seq_num + n_seq_nums * (
               convert_day_obs_to_ordinal(day_obs, day_obs_begin)
               + n_days * (
                  controller_id
                  n_controllers * is_one_to_one_reinterpretation
               )
         )
      )
   )

with the following definitions:

``detector``
   An integer detector ID (0-204, inclusive, for LSSTCam; 0-8 for LSSTComCam, always 0 for LATISS).
``n_detectors``
   The number of detectors for this instrument (256, for all instruments; rounds up the maximum to the nearest power of two).
``seq_num``
   The same per-night counter as in the decimal-digit concatenated IDs.
``n_seq_nums``
   A cap on the number of exposures per night (32768; conservatively allows for an ``exposure`` to be taken every 2.6s for 24h).
``convert_day_obs_to_ordinal()``
   N function that converts the ``day_obs`` value to the number of ordinal days since ``day_obs_begin``.
``day_obs_begin``
   The first ``day_obs`` for which this packing scheme is valid (``2010-01-01``).
``n_days``
   The number of ordinal days over which this packaging schema is valid (16384; about 45 years).
``controller_id``
   Unused, but potentially an integer corresponding to the controller used (``0`` for the default Observatory Control System (OCS) used for all science data; ``1`` for the `CCS``).
``n_controllers``
   The number of controller to allocate space for.
   Set to ``1`` to assume all Tata is from the OCS.
``is_one_to_one_reintepretation``
   ``1`` if this is a ``visit`` defined by reinterpreting the first ``exposure`` in a multi-snap sequence of observations as a single, one-to-one ``visit``, ``0`` otherwise (i.e. the representation of the rate leading ``9`` in :ref:`observation-ids`).

This logic is implemented by the :py:class:`lsst.obs.lsst.RubinDimensionPacker` class.
While the configuration of this packing scheme uses powers of two for various constants, it is implemented by actually multiplying integers, and hence its bit layout depends on the endianness of the context in which it appears.

.. _coadd-image-ids:

Coadd Image IDs
===============

Coadded images and other data products derived from coadds are identified by a ``{skymap, tract, patch}`` tuple.
A ``skymap`` is a system (identified by a string name) for apportioning the sky into slightly-overlapping regions called ``tracts``, which are then further split up into slightly-overlapping ``patches``.
All ``patches`` within a ``tract`` share a single map projection (usually gnomonic).
``Tracts`` and ``patches`` are identified by simple integer counters that are only unique within a single ``skymap`` and ``tract``, respectively.

A single integer ID for a ``{tract, patch}`` combination is computed by the following pseudocode::

   packed = patch_id * (n_patches * tract_id)

For the ``lsst_cells_v1`` skymap used in DP1, ``n_patches = 10×10 = 100``.
Note that while there is typically one coadd for each band, we do not typically need an integer ID that packs the band in as well.

.. _source-and-object-ids:

Source and Object IDs
=====================

IDs for Sources (single-visit or single-visit difference detections) and Objects (detections on coadds) are formed by packing a per-detector or per-patch counter into the image IDs described in :ref:`packed-detector-observation-ids` and :ref:`coadd-image-ids`, respectively, via::

   packed = counter + (n_counters * (image_id + (n_images * release_id)))

With the following definitions:

``counter``
   The per-detector or per-patch counter mentioned above.
``n_counters``
   A cap on the number of detections in an image.
   This is set such that the full ID fits within 63 bits (i.e. a signed integer 64, but always positive).
   For Source IDs, this is ``524288`` (just over one Source every 6×6 pixels).
   For Object IDs using the ``lsst_cells_v1`` skymap, this is ``68719476736`` (very conservative; this is approximately 6000 Objects *per pixel*).
``image_id``
   The per-detector (for Sources) or per-patch (for Objets) image ID.
``n_images``
   The maximum value for ``image_id``, rounded up to the nearest power of two.
   For Sources this is ``274877906944``.
   For Objects with ``lsst_cells_v1`` it is ``2097152``.
``release_id``
   An integer ID that identifies the data release.
   Development versions of the pipelines use ``release_id=0``.
   The ``release_id`` for DP1 is ``4``.
   This will also be the value used by non-official processing with the ``LSSTComCam`` DRP pipeline on the ``v29`` release branch, in order to make it possible to exactly reproduce the official processing.
   Space is reserved in this scheme for 64 distinct releases.

This logic is implemented by the :py:class:`lsst.meas.base.IdGenerator` class.
