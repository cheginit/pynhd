.. image:: https://raw.githubusercontent.com/cheginit/hydrodata/develop/docs/_static/pynhd_logo.png
    :target: https://github.com/cheginit/pynhd
    :align: center

|

.. image:: https://img.shields.io/pypi/v/pynhd.svg
    :target: https://pypi.python.org/pypi/pynhd
    :alt: PyPi

.. image:: https://img.shields.io/conda/vn/conda-forge/pynhd.svg
    :target: https://anaconda.org/conda-forge/pynhd
    :alt: Conda Version

.. image:: https://codecov.io/gh/cheginit/pynhd/branch/master/graph/badge.svg
    :target: https://codecov.io/gh/cheginit/pynhd
    :alt: CodeCov

.. image:: https://github.com/cheginit/pynhd/workflows/build/badge.svg
    :target: https://github.com/cheginit/pynhd/workflows/build
    :alt: Github Actions

.. image:: https://mybinder.org/badge_logo.svg
    :target: https://mybinder.org/v2/gh/cheginit/hydrodata/develop
    :alt: Binder

|

.. image:: https://www.codefactor.io/repository/github/cheginit/pynhd/badge
   :target: https://www.codefactor.io/repository/github/cheginit/pynhd
   :alt: CodeFactor

.. image:: https://img.shields.io/badge/code%20style-black-000000.svg
    :target: https://github.com/psf/black
    :alt: black

.. image:: https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white
    :target: https://github.com/pre-commit/pre-commit
    :alt: pre-commit

|

🚨 **This package is under heavy development and breaking changes are likely to happen.** 🚨

Features
--------

PyNHD is a part of `Hydrodata <https://github.com/cheginit/hydrodata>`__ software stack
and provides an interface to access
`WaterData <https://labs.waterdata.usgs.gov/geoserver/web/wicket/bookmarkable/org.geoserver.web.demo.MapPreviewPage?1>`__
and `NLDI <https://labs.waterdata.usgs.gov/about-nldi/>`_ web services. These two web services
can be used to navigate and exctract vector data from NHDPlus V2 database such as
catchments, HUC8, HUC12, GagesII, flowlines, and water bodies. Additionally, PyNHD
has some extra utilities for processing flowlines:

- ``prepare_nhdplus``: For cleaning up the dataframe by, for example, removing tiny networks,
  adding a ``to_comid`` column, and finding a terminal point if it doesn't exists.
- ``topoogical_sort``: For sorting the river network topologically which is useful for routing
  and flow accumulation.
- ``vector_accumulation``: For computing flow accumulation in a river network. This function
  is generic and any routing method can be plugged in.

Moreover, requests for additional functionalities can be submitted via
`issue tracker <https://github.com/cheginit/pynhd/issues>`__.


Installation
------------

You can install PyNHD using ``pip`` after installing ``libgdal`` on your system
(for example, in Ubuntu run ``sudo apt install libgdal-dev``):

.. code-block:: console

    $ pip install pynhd

Alternatively, PyNHD can be installed from the ``conda-forge`` repository
using `Conda <https://docs.conda.io/en/latest/>`__:

.. code-block:: console

    $ conda install -c conda-forge pynhd

Quickstart
----------

Let's explore capabilities of ``NLDI``. We need an instantiate the class first:

.. code-block:: python

    from pynhd import NLDI

    nldi = NLDI()
    station_id = "USGS-01031500"
    UT = "upstreamTributaries"
    UM = "upstreamMain"

We can get the basin geometry for the USGS station number 01031500:

.. code-block:: python

    basin = nldi.getfeature_byid("nwissite", station_id, basin=True)

``NLDI`` offer navigating a river network from any point in the network in the
upstream or downstream direction. We can limit the navigation distance (in meters). The navigation
can be done for all the valid NLDI sources which are ``comid``, ``huc12pp``, ``nwissite``,
``wade``, ``WQP``. For example, let's find all the USGS stations upstream of  01031500,
accross the whole network (tributaries) and then only in the mail channel.

.. code-block:: python

    args = {
        "fsource": "nwissite",
        "fid": station_id,
        "navigation": UM,
        "source": "nwissite",
        "distance": None,
    }

    st_main = nldi.navigate_byid(**args)

    args["navigation"] = UT
    st_trib = nldi.navigate_byid(**args)

    args["distance"] = 100
    st_d100 = nldi.navigate_byid(**args)

We can set the source to ``huc12pp`` to get HUC12 pour points.

.. code-block:: python

    args.update{
        "distance": None,
        "source" : "huc12pp",
    }
    pp = nldi.navigate_byid(**args)

``NLDI`` provides only ``comid`` and geometry of the flowlines which can further
be used to get the other available columns in the NHDPlus databse. Let's see how
we can combine ``NLDI`` and ``WaterData`` to get the NHDPlus data for our station.

.. code-block:: python

    wd = WaterData("nhdflowline_network")

    args.update{
        "fsource": "comid",
        "source" : None,
        "navigation": UM,
    }
    comids = nldi.navigate_byid(**args).nhdplus_comid.tolist()
    flw_main = wd.byid("comid", comids)

    args["navigation"] = UT
    comids = nldi.navigate_byid(**args).nhdplus_comid.tolist()
    flw_trib = wd.byid("comid", comids)

Other feature sources in the WaterData database are ``nhdarea``, ``nhdwaterbody``,
``catchmentsp``, ``gagesii``, ``huc08``, ``huc12``, ``huc12agg``, and ``huc12all``.
For example, we can get the contributing catchments of the flowlines using ``catchmentsp``.

.. code-block:: python

    wd = WaterData("catchmentsp")
    catchments = wd.byid("featureid", comids)

The ``WaterData`` class also has a method called ``bybox`` to get data from the feature
sources within a bounding box.

.. code-block:: python

    wd = WaterData("nhdwaterbody")
    wb = wd.bybox((-69.7718, 45.0742, -69.3141, 45.4534))

Next, lets clean up the flowlines and use it to compute flow accumulatin. For simplicity,
we assume that the flow in each river segment is equal to the length of the segment. Therefore,
the accumulated flow at each point should be equal to sum of the lengths of all its upstream
river segments i.e., ``arbolatesu`` column in the NHDPlus database. We can use this to validate
the flow accumulation result.

.. code-block:: python

    import pynhd as nhd

    flw = nhd.prepare_nhdplus(flw_trib, 1, 1, 1, True, True)

    def routing(qin, q):
        return qin + q

    qsim = nhd.vector_accumulation(
        flw[["comid", "tocomid", "lengthkm"]], routing, "lengthkm", ["lengthkm"],
    )
    flw = flw.merge(qsim, on="comid")
    diff = flw.arbolatesu - flw.acc

    print(diff.abs().sum() < 1e-5)

Contributing
------------

Contirbutions are very welcomed. Please read
`CODE_OF_CONDUCT.rst <https://github.com/cheginit/pynhd/blob/master/CODE_OF_CONDUCT.rst>`__
and
`CONTRIBUTING.rst <https://github.com/cheginit/pynhd/blob/master/CONTRIBUTING.rst>`__
files for instructions.