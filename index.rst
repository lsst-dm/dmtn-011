..  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
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
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

This document summarizes the work that was done to test the accuracy of shear measurement using the measurement algorithms in the DM package meas_modelfit.git.  This package contains the Python package lsst.meas.modelfit, which houses both the CModel algorithm -- used to measure galaxy shapes, and the ShapeletPsfApprox Algorithm (SPA) -- used to create a parameterized model of the point spread function (Psf).

The images used for this test were created by GalSim simulations as described below. The galaxy profiles areies parameterized GalSim models. However, the Psfs are not from GalSim models, as we want the framework to accommodate a variety of different realistic and simulated Psfs. A separate psf image for each galaxy is used to simulate the effects of seeing:  this includes both atmospheric and optical effects. 

In these tests, the Psf is modelled using SPA for later use during any measurement process where an estimation of the Psf is required (for example, for Psf weighting).

Our goal is to measure enough simulated galaxies to determine the effect of different Psf approximation and shape measurement algorithms on galaxy shear estimation.

DM-5447a – Techniques used in this simulations
=================================================

Creating the Psf Library:
-------------------------
We used PhoSim to create Psf images for an entire LSST focal plane.  The LSST focal plane is divided into 21 rafts with 9 sensors per raft.  To create a library of Psf images, 10000 positions were selected at random and PhoSim was used to create a stellar image at each position.  After removing unusable stars (those which did not fall fully on any particular sensor, or which were close to another star), between 7000 and 8000 usable stellar images remained for each simulated focal plane.

Psf libraries were created for “raw seeing” values of 0.5, 0.7, and 0.9 arcseconds and for LSST filters (f2 and f3).  Most of the tests during this cycle were run using the f2_0.7 library.

Using Great3Sims:
-------------------------

We used the Great3sims package to produce yaml files which in turn drive the GalSim creation of galaxy images.  Great3Sims produces a catalog of galaxies to be constructed using parameterized models and randomly drawn parameters.  A constant shear is then applied for each “subfield”, with the shear and shear angle drawn at random from a specified range.  In this respect, our galaxy sample was the same as that used for the “control” branch of the great3sims test.

One important modification to Great3Sims was to allow the point spread function for each galaxy to be sampled for our Psf Library rather than from a parameterized Psf model.  The PhoSim Psf libraries described above were used to select a Psf image  GalSim during the creation of the galaxy images.  So instead of convolving the test galaxies with a parameterized Psf model, we supplied the Psf for each model as a fits image (typically 67x67).  The fits image was supplied to GalSim using the InterpolatedImage input type.

The great3sims “control/ground/constant” tests provided for 200 subfields, each with a constant shear and shear angle.  There are 10000 galaxies per subfield.  Each subfield is assigned its own shear and shear angle at random.

In some of our early tests, we made the subfields smaller (1024 galaxies), but made the number of subfields larger (1024 subfields for each shear value).  This was done in an attempt to sample the error in the absolute value of the shear.  Some of these early tests were also larger (6 million galaxies), which made our tests more sensitive, but possibly not realistic.

Using GalSim:
-------------------------

GalSim is driven by the Great3sims program, which creates a yaml file for GalSim. The profile and shear information if supplied for each galaxy through epoch catalog.  Except for the introduction of a Psf image, we did not alter this relationship between Great3sims and GalSim.

The epoch catalog is typically named epoch_catalog-nnn-m.fits, with nnn the ordinal number of the subfield, and m the epoch (always 0 in our tests).  GalSim creates a matching image-nnn-m.fits with a 96x96 pixel image for each galaxy in the sample.

Doing Measurements:
-------------------------

The goal of the framework is to compare the results of a shape measurement algorithm with the known constant shear values which are produced with the Great3sims/GalSim simulation.  The algorithm must be housed in an lsst.meas.base measurement plugin so that it can be used by the LSST DM Stack.

Our intial tests were with the CModel algorithm in lsst.meas.modelfit package, but this framework could be used to test any shape algorithm which is housed in a measurement plugin.

Shear Bias Analysis:
====================

We wish to estimate the multiplicative and additive biases of our algorithm for both components of shear. For each shear component, this is done by plotting the measured shear against the applied shear from the epic catalog.  The multiplicative bias is the slope of our regression line, and the additive bias is the intercept.

  .. figure:: /_static/figure_1.png
     :name: figure_1
     :target: _images/figure_1.png

     Graph of regression line fit to 200 subfield measurements

In this example, the regression lines lie very close to each other, with the measured shear multiplicatively biased by about +50%.  The intercepts deviate from zero by 7e-4 and 2e-4 respectively.

Comparison of Shear Bias for Different Algorthms:
-------------------------------------------------

To see how different algorithms measure up against each other, we plot their shear biases against each other.  As in all of these tests, we take what we regard as a preferred measurement, in this case a 7th order fits of the Psf, against a easier but possibly less accurate measure: n this plot, three lower order estimations are shown. We assume the preferred measurement to be correct, and look for whether either the multiplicative or additive bias of the test measurement differs from the preferred measurement by a significant amount.

This idea of using one of the two algorithms as a measuring stick for the others originally started when we were testing algorithms which took a lot of computing time against “shortcuts”, which were clearly not as accurate but required less time.  For example, one of our tests was to compare measurements which used the full galaxy cutout (96x96) against measurements which used only a part of the cutout (24x24).  The goal in those test was to see whether a smaller cutout would make any difference, and using the full cutout as a measuring stick makes sense.

  .. figure:: /_static/figure_2.png
     :name: figure_2
     :target: _images/figure_2.png

     Comparison of shear bias for different algorithms

One  question frequently asked is how this relates to the LSST requirement that we achieve a 3e-3 accuracy in the measurement of the slope, and 5e-4 in intercept.  If we assume that any algorithm can be accurately calibrated, these our errorbars would presumably represent the errors in the calibrated bias.  However, our tests do not tell us how well we meet this requirement.  The size of the error bars certainly indicate something about the resolution of our measurements.  But we don't actually have a way to calibrate our calculation of shear bias.  That depends on the fidelity of the simulations

How errors were determined:
---------------------------

In the tests discussed below, we really ran only two simulations.  One was the set of 200 subfields, each subfield having 10000 galaxies with the same applied shear.  The other was a set of 6x1024 subfields at 6 values of total shear, with 1024 galaxies at each rotation angle.

Looking forward, I would like to run more random simulations which give us a better idea of how our shear biases vary from simulation to simulation.

We calculated our errors in two ways, one using estimators of the errors from each subfield to analytically determine the errors in the slope and intercept measurements. The other was to use a bootstrap technique to see how the bias varied within resampled distributions.

1.  The analytic approach
^^^^^^^^^^^^^^^^^^^^^^^^^

If (e1,e2) is the measure of the galaxy ellipticity and (g1,g2) is the applied shear, the deviation of (e1-g1,e2-g2) from (0,0) is a measure of our error.  Of course, this measurement is dominated by shape noise. By averaging e1 and e2 over and entire subfield, we get an estimator of the applied shear, as well as the error in this estimation.

Since the regression to determine m1, c1, m2 and c2 uses as its data the (e1avg, e2avg)  vs. (g1, g2), we can use the errors in e1avg and e2avg to determine the errors in the regression values.  See Numerical Recipes section 15.2 for a description of this calculation.

2.  Bootstrap estimation
^^^^^^^^^^^^^^^^^^^^^^^^^

However, since we did not entirely trust the analytic approach, we attempted to determine the error in our shear bias calculation by running our entire simulation 100-300 times using a bootstrap technique.  That is, we did each subfield measurement again (for each of the 200 subfields) with 5000 galaxy pairs, sampled from the original 5000 galaxy pairs with replacement.  The bootstrap method consistently gives an error which is larger than the analytic approach by a factor of 2.1-2.3.

Since the bootstrap estimation indicates a larger error, we decided to increase our errors uniformly by a factor of 2.2 for this study.  It would obviously be even better to examine the errors over a large number of simulations.  20 simulations with 2 million galaxies each would probably give us a better handle on both the error in the shear biases, and also an absolute calibration of the shear measurements.



Testing Parameterizations of ShapeletPsfApprox (DM-1136 and DM-4214)
========================

These are a set of tests which are intended to test ShapeletPsfApprox(SPA), and in particular, to find the shapelet order above which there is little additional value.  SPA is a four component “multi-shapelet function” model of the Psf, where the four components from narrowest to broadest are named “inner”, “primary”, “wings”, and “outer”.  An SPA model is designated in this document with four numbers:  for example, 1234 means inner order = 1, primary order = 2, wings order = 3, and outer order = 4.

Test setup
--------------

This set of tests is run using the great3sims configuration of 200 subfields with 10000 galaxies each.  All were run with a single fixed Psf with filter f3 and raw seeing 0.7 arcsec.  The errors shown below are estimated using bootstrap resampling, as discussed in the introduction.  Please note that most of the differences in parameterizations were washed out when the errors were increased by x2.2.  So getting a better estimate of the errors could be important to these results.

Finding the best parameterization is a little tricky, as the effects of the parameters are not independent of each other.  To make this comparison more tractable, I've chose to take slices through the parameter space, relying to some extent on the knowledge that the “primary” parameter has the biggest effect, followed by the “wings”.  The “inner” and “outer” seem to have less of an effect.  

My first set was to compare parameterizations of the form 0nn0, where n=2,3,4,5,6.  The 3773 parameterization is shown for reference.

  .. figure:: /_static/figure_4.png
     :name: figure_4
     :target: _images/figure_4.png

     Parameterizations of the form 0nn0.  Full is 0220

.. literalinclude:: /_static/table_1.txt

Things appear to settle around 0440 or 0550.  But none of these differ from the reference model by 3 sigma.

However, note the similar comparisons with inner and outer set to 2 or 3. The primary and wings settings seem to settle by order 4 with higher order in the inner and outer components.


  .. figure:: /_static/figure_5.png
     :name: figure_5
     :target: _images/figure_5.png

     Parameterizations of the form 1nn1.

  .. figure:: /_static/figure_6.png
     :name: figure_6
     :target: _images/figure_6.png

     Parameterizations of the form 2nn2.

  .. figure:: /_static/figure_7.png
     :name: figure_7
     :target: _images/figure_7.png

     Parameterizations of the form 3nn3.

Here is a full set of comparisons against 3773 where primary and wings are the same, and inner and outer are the same.  They suggest that order 3 may be nearly as good as the higher orders, with improvement in the Psf modeling appearing to max out between order 3 and 4.

.. literalinclude:: /_static/table_2.txt

Conclusion:
----------

It is hard to draw too many hard conclusions from these studies. For one thing, the models are close enough to each other above order 3 that the accuracy of our error measurement becomes very important. If the 2.2 multiplier which we applied (consistent with the bootstrap resampling) is correct, then orders above 4 not a useful improvement with the 2 million galaxies in this test.

In future studies, we should resample from a larger number of galaxies, so the errors become much more obvious.  Varying the Psf from point-to-point on the focal plane may also have an effect which we have not studied in this set of tests.
