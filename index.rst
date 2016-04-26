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

ThesShear measurement was done on images created by GalSim simulations. The galaxy profiles are parameterized GalSim models. However, the Psfs are not parameterized GalSim models, as we wanted the framework to be able to accommodate a variety of different realistic and simulated Psfs. A separate psf image for each galaxy is used to simulate the effects of seeing, including both atmospheric and optical effects. 

In these tests, the Psf is modelled using SPA for later use during any measurement process where an estimation of the Psf is required (for example, for Psf weighting).

Our goal is to measure enough simulated galaxies to determine the effect of different Psf approximation and shape measurement algorithms on galaxy shear estimation.

DM-5447a – Techniques used in this simulations
=================================================

Creating the Phosim Psf Library:
-------------------------
PhoSim is an LSST simulator which is used create images of galaxies and stars over the entire LSST focal plane. This simulation includes variations in the atmosphere and the optics (telescope and camera). Please see https://confluence.lsstcorp.org/pages/viewpage.action?pageId=4129126 for a description of this software.

We used PhoSim to create stellar images scattered at random over the LSST focal plane. The LSST focal plane is divided into 21 rafts with 9 sensors per raft.  To create a library of Psf images, 10000 positions were selected at random and PhoSim was instructed to create a stellar image at each position.  After removing unusable stars (those which did not fall fully on any particular sensor, or which were close to another star), between 7000 and 8000 usable stellar images remained for each simulated focal plane.

Psf libraries were created for “raw seeing” values of 0.5, 0.7, and 0.9 arcseconds and for LSST filters (f2 and f3).  Most of the tests during this cycle were run using the f2_0.7 library.  

Using Great3Sims:
-------------------------
The great3-public repository (https://github.com/barnabytprowe/great3-public) provided the scripts for selecting a sample of galaxies. This packages creates a catalog of galaxies drawn at random from a distribution of galaxies, and allows us to select a Psf and applied shear for each galaxy.  The output of these scripts is the combination of an "epoch catalog" and a yaml file. The epoch catalog have one line per galaxy, describing the galaxy characteristics and the applied Psf and shear.  The yaml file is used to drive GalSim to make images.

Test Setup A:
^^^^^^^^^^^^
The original Great3Sims default configuration produced a test of 2 million galaxies (see Setup B). In some of our early tests, the subfields were smaller (1024 galaxies) and of constant shear angle. The shear angle was then varied by creating a large number of subfield with different shear angles. 

The absolute value of the applied shear was varied in 6 steps from .0001 to .05.

These tests varied in size, from as few as 128 subfields to as many as 2048. 

Test Setup B:
^^^^^^^^^^^^
This galaxy sample was the same as that used for the control branch of the Great3sims tests. The Great3sims "control/ground/constant�" branch creates images for 200 subfields, each with a constant shear and shear angle.  There are 10000 galaxies in each subfield. 

This setup has 2 million galaxies, and is basically a single sample which was used for all of our comparisons.  However, it is possible to modify this test for different Psf assumptions.  One difference is in the seeing conditions (0.5, 0.7, and 0.9 arcseconds) and filter (f2 or f3) used by PhoSim.  Another variation is to either hold the Psf constant over the entire focal plane, or allow it to vary according to atmosphere and telescope optics.

One important modification to Great3Sims was to allow the point spread function for each galaxy to be sampled from our PhoSim Library described above. These Psf images were typically 67x67 pixels at the LSST plate scale. The fits image was given to GalSim using the InterpolatedImage input type.

Using GalSim:
-------------------------
See http://galsim-developers.github.io/GalSim for information about GalSim and how it is used. As discussed above, the Great3Sims package supplies the galaxies parameters in an epoch catalog, as well as a matching yaml file which feeds those parameters to GalSim for image construction. Our only major change was to supply the PhoSim Psf to GalSim as an image. 

The epoch catalog is typically named epoch_catalog-nnn-m.fits, with nnn the ordinal number of the subfield, and m the epoch (always 0 in our tests).  GalSim creates a matching image-nnn-m.fits with a 96x96 pixel image for each galaxy in the sample.

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

To see how different algorithms measure up against each other, we plot their shear biases against each other. In these tests we take what we regard as a preferred measurement, in this case a measurement using the entire galaxy cutout (64x64) again two measurements done with much smaller cutout. We assum that the 64x64 measuremnt is the best we can do, and look to see if the multiplicative or additive bias of the other measurements differs from the preferred measurement by a significant amount.

This idea of using one of the algorithms as a measuring stick for the others can tell us whether it makes any difference to take the additional computing time to do the larger image.  In this test, the (40x40) cutout is significantly different than the (24x24) cutout.  However, it does not differ from the (64x64) cutout.

  .. figure:: /_static/figure_2.png
     :name: figure_2
     :target: _images/figure_2.png

     Comparison of shear bias for different algorithms

One  question frequently asked is how this relates to the LSST requirement that we achieve a 3e-3 accuracy in the measurement of the slope, and 5e-4 in intercept.  If we assume that any algorithm can be accurately calibrated, these our errorbars would presumably represent the errors in the calibrated bias.  However, our tests do not tell us how well we meet this requirement.  The size of the error bars certainly indicate something about the resolution of our measurements.  But we don't actually have a way to calibrate our calculation of shear bias.  That depends on the fidelity of the simulations

How errors were determined:
---------------------------

In the tests discussed below, we really ran only two simulations.  One was the set of 200 subfields, each subfield having 10000 galaxies with the same applied shear, or about 2 million galaxies in all. However, the large number of galaxies are really needed just to beat down the shot noise. Running 10 or even 20 samples of this size and measuring the variation between samples would be a useful technique for estimating the errors in our shear biases measurements.


However, for this study, we did not run multiple instances of the 2 million galaxy simulation, and we calculated our errors by attempting to look at the variation within the shear measurements in each subfield. We calculated our errors in two ways. One was to use estimators of the errors from each subfield to analytically determine the errors in the slope and intercept measurements. The other was to use a bootstrap technique to see how the bias parameters varied within resampled distributions.

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

Test Setup B (2 million galaxies)

Psf: filter f2 and "raw seeing" 0.7 arcsecs

Single Psf over the focal plane

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

Test Setup A (1-12 million galaxies) at 6 values of shear.

Psf: filter f3 and "raw seeing" 0.7 arcsecs.  Psf varied over the focal plane (all 21 rafts x 9 sensors/raft)

Error estimation increased by as indicated by subfield variation. No bootstraps were run.

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

nInitialRadii Test:
^^^^^^^^^^^^^^^^^^^

In this test, the nInitialRadii configuration of CModel was varied from 0 to 15.
(add reference to initial test DM-3376 and later test DM-????)

Stampsize Test:
^^^^^^^^^^^^^^^^^^^

In this test, the galaxy stampsize was varied from 20 to 64.  The original galaxy postage stamps created by GalSim were 96x96 at the LSST plate scale.  The stampsize test was to feed CModel smaller postage stamps by trimming the edges of each galaxy image.
(add reference to test DM-1135 and later test DM-????)
