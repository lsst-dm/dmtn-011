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

This document describes a framework which was created to test how well shape measurement algorithms in the DM Stack can measure galaxy shear.  The targets algorithms for this framework are in the package meas_modelfit. This package houses both the CModel algorithm -- used to measure galaxy shapes) and the ShapeletPsfApprox Algorithm (SPA) -- used to create a parameterized model of the Psf (point spread function). However, the framework is set up to allow similar tests on other shape measurment algorithms.

Shear measurement was done on images created by Great3Sims and GalSim as described below.  However, the Point Spread Function (Psf) which was applied to the simulated galaxies was created by PhoSim using an LSST simulation.

Our goal was to run both Psf approximation and shape measurement algorithms on large numbers of galaxies that have been subjected to known constant shear and Psf distortions, and to characterize the shear bias of these algorithms.

Techniques used in this simulations
=================================================

Using Great3Sims: 
-------------------------
The great3-public repository (https://github.com/barnabytprowe/great3-public) provided the scripts for creating catalogs of galaxies. This package draws galaxy parameters at random from a known distribution, and selects a Psf and applied shear for each galaxy.  The outputs are an "epoch catalog" and a yaml file. The epoch catalog has one line per galaxy, and is used as the "truth" value in our later tests. The yaml file is fed to GalSim to create grids of postage stamps (in fits format) of the galaxies.

Branch Setup:
^^^^^^^^^^^^
Galaxy samples were created using the Great3sims "control/ground/constant" branch. Each subfield has a single constant shear and shear angle. The galaxies are created in pairs rotated at 90 degrees.  All galaxies in a subfield are subjected to the same applied shear.  

One important modification to Great3Sims was to sample the Psf for each galaxy from a PhoSim Psf Library (described below). The Psf images were typically 67x67 pixels at the LSST plate scale.

Using GalSim:
-------------------------
See http://galsim-developers.github.io/GalSim for information about GalSim and how it is used. As discussed above, the Great3Sims package supplies an epoch catalog for each subfield, as well as a matching yaml file which feeds the galaxy parameters to GalSim.  The result is an image of galaxies on a grid which can be used to make cutouts for measurement.

  .. figure:: /_static/sample_image-000-1.png
     :name: sample_images
     :target: _images/sample_image-000-1.png

     Galaxy images created by GalSim (this is a 4x4 galaxy subfield)

The epoch catalog is named epoch_catalog-nnn-m.fits, with nnn the ordinal number of the subfield, and m the epoch (always 0 in our tests).  GalSim creates a matching image-nnn-m.fits with a 96x96 pixel image for each galaxy in the sample.

Creating the Phosim Psf Library:
-------------------------
PhoSim is a ray tracing simulator which is used create images of galaxies and stars. This simulator has a description of the LSST telescope and camera, and will create simulated images with distortions due to variations in the atmosphere and optics. Please see https://confluence.lsstcorp.org/pages/viewpage.action?pageId=4129126 for a description of this software.

We used PhoSim to create stellar images scattered at random over the LSST focal plane. The LSST focal plane is divided into 21 rafts with 9 sensors per raft.  To create a library of Psf images, 10000 positions were selected at random and PhoSim was instructed to create a stellar image at each position.  After removing unusable stars (those which did not fall fully on any particular sensor, or which were close to another star), between 7000 and 8000 usable stellar images remained for each Psf library.

Psf libraries were created for "raw" seeing values of 0.5, 0.7, and 0.9 arcseconds through LSST filters (f2 and f3). The seeing is actually somewhat worse after all the simulated distortions are applied. 

Most of the tests during this cycle were run using the f3_0.7 library.  

Doing Measurements:
-------------------------

The goal of the framework is to compare the results of a shape measurement algorithm with the known constant shear values which stored in the epoch catalog.  The algorithm must be housed in an lsst.meas.base measurement plugin so that it can be used by the LSST DM Stack. And it must produce some measurement, such as second moments, which can be used to calculate galaxy ellipticity.

Our tests in this study were with the CModel algorithm in lsst.meas.modelfit package, but this framework could be used to test any shape algorithm which is housed in a measurement plugin.

Shear Bias Analysis:
====================

We wish to estimate the multiplicative and additive biases of our algorithm for both components of shear. For each shear component, this is done by plotting the measured shear against the applied shear from the epoch catalog.  The multiplicative bias is the slope of our regression line, and the additive bias is the intercept.

  .. figure:: /_static/figure_1.png
     :name: figure_1
     :target: _images/figure_1.png

     Graph of regression line fit to 200 subfield measurements

In this example, the regression lines lie very close to each other, with the measured shear multiplicatively biased by about +50%.  The intercepts deviate from zero by 7e-4 and 2e-4 respectively.

Comparison of Shear Bias for Different Algorthms:
-------------------------------------------------

To see how different algorithms measure up against each other, we plot their shear biases against each other. In these tests we take what we regard as a preferred measurement, in this case a measurement using a high order fit of the Psf (3773) against lower order fits: SingleGaussian, DoubleGaussian, and Full. These are the default parameterizations of the meas_modelfit algorithm ShapeletPsfAlgorithm, which we shall explore in more detail later in this document.

In this graph, we see how the Single and Double Gaussian fits of the Psf are very different than the high order fit, 3773.  The parameterization "Full", however is pretty close to the 3773 algorithm in all four shear bias values.

  .. figure:: /_static/figure_3.png
     :name: figure_3
     :target: _images/figure_3.png

     Comparison of shear bias for different algorithms

How errors were determined:
---------------------------

In the tests discussed below, we really ran only one simulation for each test. 
However, the large number of galaxies are really needed just to beat down the shot noise. Running 10 or even 20 samples of this size and measuring the variation between samples would be a useful technique for estimating the errors in our shear biases measurements.


Wwe did not run multiple instances of the 2 million galaxy simulation, and we calculated our errors by attempting to look at the variation within the shear measurements in each subfield. We calculated our errors in two ways. One was to use estimators of the errors from each subfield to analytically determine the errors in the slope and intercept measurements. The other was to use a bootstrap technique to see how the bias parameters varied within resampled distributions.

1. Estimating the error of the mean in each subfield. 
^^^^^^^^^^^^^^^^^^^^^^^^^

If (e1,e2) is the measure of the galaxy ellipticity and (g1,g2) is the applied shear, the deviation of (e1-g1,e2-g2) from (0,0) is a measure of our error.  Of course, this measurement is dominated by shape noise. By averaging e1 and e2 over and entire subfield, we get an estimator of the applied shear, as well as the error in this estimation.

Since the regression to determine m1, c1, m2 and c2 uses as its data the (e1avg, e2avg)  vs. (g1, g2), we can use the errors in e1avg and e2avg to determine the errors in the regression values.  See Numerical Recipes section 15.2 for a description of this calculation.

2.  Bootstrap estimation
^^^^^^^^^^^^^^^^^^^^^^^^^

However, since we did not entirely trust the approach under (1), we attempted a second approach using bootstrapping to produce different resampled populations of galaxies from out single set of 2 million galaxies. This allowed us to do the shear bias calculation multiple times and observe the variation in the shear parameters on these different populations. The great3sims subfields are actually 5000 galaxy pairs, so our resampling was actually done by drawing from each sample of 5000 with replacement.

The bootstrap method consistently gives an error which is larger than the analytic approach by a factor of 2.1-2.3.

Since the bootstrap estimation indicates a larger error, we decided to increase our errors uniformly by a factor of 2.2 for this study.

Testing Parameterizations of ShapeletPsfApprox (DM-1136 and DM-4214)
========================

These are a set of tests which are intended to test ShapeletPsfApprox(SPA), and in particular, to find the shapelet order above which there is little additional value.  SPA is a four component “multi-shapelet function” model of the Psf, where the four components from narrowest to broadest are named “inner”, “primary”, “wings”, and “outer”.  An SPA model is designated in this document with four numbers:  for example, 1234 means inner order = 1, primary order = 2, wings order = 3, and outer order = 4.

Test setup
--------------

Subfields: 10000 galaxies/subfield in 5000 pairs, all with the same applied shear and shear angle
 
          200 subfields total (2 million galaxies) at a variety of shear and shear angles

Psf: filter f2 and "raw seeing" 0.7 arcsecs. Single Psf selected from f2_0.7 PhoSim library.

Error estimation increased by x2.2 as indicated by bootstrap tests.


Finding "best" parameterization for ShapeletPsfApprox.
--------------
We are hoping to find out how high the order of the estimation parameters must be before our measurement of the bias parameters plateaus.

Comparing parameterization is a little tricky, as the effects of the parameters are not independent of each other.  To make this comparison more tractable, I've chose to take slices through the parameter space, relying to some extent on the knowledge that the “primary” parameter has the biggest effect, followed by the “wings”.  The “inner” and “outer” seem to have less of an effect.  

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

However, it does appear that the order 2 parameterizations are significantly different from 3773, and that order 3 is needed at least in the primary and wings modelling. But there isn't all that much to choose from above order 3.  In fact, 2332 seems to be quite close to 3773, and 3333 is even a little better.

It is obvious that a lot depends on our error assumptions. In future studies, we should create several populations of the same setup to make true variation easier to determine.  Varying the Psf from point-to-point on the focal plane may also have an effect which we have not studied in this set of tests.

Testing CModel Configurations
========================
Our earliest tests were tests of the CModel algorithm, leaving the SPA for Psf estimation at its defaults.  CModel is a shape measurement algorithm in the meas_modelfit package of the DM Stack. This study was done partly to see the effects varying the CModel task config, and partly to determine how large a galaxy sample was required.

As a result of these these tests, we decided to move our tests for single host, multi-core machines at Davis to the computing farm at SLAC. 

Test setup
--------------
Subfields: 1000 galaxies/subfield in 500 pairs, all with the same applied shear and shear angle
           The applied shear was varied in 6 steps from .0001 to .05.

The shear angle was selected randomly from 0-360 in each of the subfields

These tests varied in size, from as few as 128 subfields to as many as 2048 (1-12 million galaxies). 

Psf: filter f3 and "raw seeing" 0.7 arcsecs.  Psf selected randomly for each galaxy

Error estimation indicated by subfield variation. No bootstraps were run.

Test Descriptions:
--------------
The goal of these tests was to demonstrate the effect of different the CModel measurement configurations on the shear bias. 

These tests were done using Test Setup A. 

Initially, the test was done with 128 subfields at 6 shear values (.0001, .01, .02, .03, .04, .06).  This is a total of about 6 x 128 x 1024 = 786432 galaxies.

Where these tests did not show enough sensitivity, tests of 1024 subfields and 2048 subfield were used (6 and 12 million galaxies).

nGrowFootprint Test:
^^^^^^^^^^^^^^^^^^^

In this test, the nGrowFootprint configuration of CModel was varied from 0 to 10.
(add reference to initial test DM-3375 and later test DM-????)

  .. figure:: /_static/nGrow.larger.png
     :name: figure_7
     :target: _images/nGrow.larger.png

     nGrowFootprint vs. shear bias

Stampsize Test:
^^^^^^^^^^^^^^^^^^^

In this test, the galaxy stampsize was varied from 20 to 64.  The original galaxy postage stamps created by GalSim were 96x96 at the LSST plate scale.  The stampsize test was to feed CModel smaller postage stamps by trimming the edges of each galaxy image.
(add reference to test DM-1135 and later test DM-????)
  .. figure:: /_static/stampsize.larger.png
     :name: figure_7
     :target: _images/stampsize.larger.png

     stampsize vs. shear bias
