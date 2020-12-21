# **Advanced Lane Lines** 

## Report by Reza Rajan

---

**Advanced Lane Lines**

The goals of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

Furthermore, this project must satisfy all rubric requirements (listed in this report). This report, in and of itself, fulfills the first requirement: 

_**Rubric Requirement**: Provide a Writeup / README that includes all the rubric points and how you addressed each one. You can submit your writeup as markdown or pdf._


[//]: # (Image References)
[distorted]: ./output_images/distorted.jpg
[undistorted]: ./output_images/undistorted.jpg
[sobel]: ./output_images/sobel.jpg
[color_mask]: ./output_images/color_mask.jpg
[colgrad_mask]: ./output_images/colgrad_mask.jpg
[perspective_transform]: ./output_images/perspective_transform.jpg
[histogram]: ./output_images/histogram.jpg
[sliding_window]: ./output_images/sliding_window.jpg
[look_ahead]: ./output_images/look_ahead.jpg
[inverse_transform]: ./output_images/inverse_transform.jpg
[final_frame]: ./output_images/final_frame_multi_sample.jpg

---

## Reflection

### Lane Detection Pipeline

To detect lane lines a pipeline is constructed such that it takes an image (frame) as input and outputs a similar image, but which has the lane lines highlighted, as well as the region occupied by the lane.

The following steps are performed in this pipeline:

1. Camera calibration
2. Distortion correction
3. Color/gradient/lighting thresholding
4. Perspective transform
5. Histogram peak extraction
6. Sliding window lane search
7. Searching from prior lane bounds (look-ahead filter)
8. Inverse perspective transform

A brief overview of the steps are outlined below:

### Camera Calibration
_**Rubric Requirement**: Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image._

This step is required prior to any lane finding, **since distortion effects of the camera will affect proper, accurate line estimates.**

The camera is calibrated using the chessboard method. In essence, multiple images of a chessboard pattern, taken from the same camera, are used to calculate the distortion coefficients and camera matrix. Once obtained, these are used to perform a perspective transform on each of the camera's images.

A helper function called `calibrate` has been created to perform this step.

---
_**Rubric Requirement**: Provide an example of a distortion-corrected image._
### Distortion Correction

With the distortion coefficients and camera matrix, the image is tranformed as shown below:

**Original Distorted Image**

![Original distorted image][distorted]

**Undistorted Image**

![Unistorted image][undistorted]

The `cv2.undistort` method is used to perform this step.

---
_**Rubric Requirement**: Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image. Provide an example of a binary image result._
### Color/Gradient/Lighting Thresholding

On the premise that lane lines are either white or yellow, are mostly vertical, and should contrast with the rest of the road, then **color, gradient and lighting may be used to distinguish lane lines from the road.**

A binary image is created by extracting yellow and white hues within certain lighting ranges from the image, using the HLS colorspace. Furthermore, the Sobel filter in the x-direction is used to extract the lane lines as vertical lines should contrast with the road.

It should be noted that a problem with this method is that lighting variations caused by shadows, for e.g. when passing under a bridge, may cause inconsistency in the thresholding. To overcome this, each image is normalized using Contrast-Limited Adaptive Histogram Equalization (CLAHE). This method is not perfect, but it helps make the pipeline more robust.

Three helper functions have been created for gradient thresholding: 
* `abs_sobel_thresh`: directional Sobel
* `mag_thresh`: magnitude Sobel
* `dir_thresh`: gradient direction Sobel

The `color_mask` helper function has been created for color thresholding.

Results of this thresholding are illustrated below:

**Sobel Gradient Thresholding Tests**

![Sobel Gradient Thresholds][sobel]

The Sobel operations indicate that the x-direction provides the most lane line filtering.

**Color Thresholding**

![Color Thresholds][color_mask]

The color mask filters for only white and yellow hues within defined lighting ranges. This uses the hue and lighting channels of the image converted from RGB to HLS. It does an excellent job at filtering for lane lines.

**Combined Thresholds**

![Combined Thresholds][colgrad_mask]

On a different image, the combined mask is shown to pick up a lot of lane line detail. Though this combined mask picks up details in the trees, this will be handled later on.

_Note: Color thresholds pick up a lot of the lane line fills, while the gradient threshold picks up edges. In areas where white lines are relatively dim, the gradient threshold recaptures these parts of the lane line. This will be apparent in the lane detection videos._

---
_**Rubric Requirement**: Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image._
### Perspective Transform

To derive metrics from the lane lines, a top-down view is required, similar to plotting lines on a graph. This is required for lane fitting and calculating the radius of curvature. 

The following four points have been used to transform the image:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 245, 720      | 450, 670      | 
| 600, 450      | 450, 50       |
| 670, 450      | 800, 50       |
| 965, 720      | 800, 670      |

Thus, using the four points the image, it is transformed using as follows:

**Perspective Transform**

![Perspective Transform][perspective_transform]

A helper function has been created for the perspective transform, called `warp`. This helper function is also used for the [inverse perspective warp](#inverse-perspective-transform) by passing the `inverse=True` parameter.

---

_**Rubric Requirement**: Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?_
### Histogram Peak Extraction

With the perspective transformed binary mask, a method called histogram peak extraction is used to identify the positions of the base of the lane lines. 

In essence, this method sums the number of white pixel for each column of the binary image. The two columns with the highest sums are taken as the base of the lane. Note that this method assumes that lane lines are, for the most part, vertical.

**Histogram Peak Extraction**

![Histogram Peak Extraction][histogram]

The `hist_peaks` helper function has been created to perform this step.

### Sliding Window Lane Search
There is now enough information to start the lane line search. The sliding window lane search divides the top-down binary image into vertically-aligned grids, i.e. windows, starting at the previously found base positions of the lane. Starting with the first window, once a set minimum number of pixels are found within the window bounds they are stored as found lane pixels, and an average is taken as the next base position for the lane search in the next window, continuing for all windows to the top of the image.

This can be though of as _shifting_ the search window along the lane. Below is an illustration of this method:

**Sliding Window Lane Search**

![Sliding Window Lane Search][sliding_window]

The `fit_polynomial` and `find_lane_pixels` helper functions are used in conjunction to perform this task.

### Searching from Prior Lane Bounds
An alternative to the sliding window lane search is searching from prior lane bounds. This method searched within a margin of previously found lane bounds, either from the sliding window lane search or recursively, from this method. This is done as a _look-ahead filter_ to improve the pipeline's performance by limiting the search bounds, and skipping the histogram peak extraction.

An illustration of this method is shown below:

**Look-Ahead Filter**

![Look-Ahead Filter][look_ahead]

The `search_poly` helper function is used to perform this task.

---
_**Rubric Requirement**: Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center._
### Calculating Metrics
With the lane line pixels found, a polynomial is fit to each line, and the radius of curvature and vehicle offset are found. These metrics are stored in a `Line()` class for both the left and right lane lines. As a further abstraction, a `Lane()` class is used to hold these two `Line()` classes, and perform sanity checks and other logic for lane estimation.

---
_**Rubric Requirement**: Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly._

### Inverse Perspective Transform
With the lane lines found and plotted, an inverse perspective transform is used, which is the inverse of the initial transformation matrix, to transform from a top-down view to a front-facing (original) view.

**Inverse Perspective Transform**

![Inverse Perspective Transform][inverse_transform]

### Video Pipeline
The lane line, along with the metrics from the `Lane()` class, are plotted onto each frame of the camera's video feed to produce the lane detection visualization. All above steps are compiled into the `pipeline` helper function, as a single function to perform the lane detection.

Samples of the final frame results are shown below:

**Final Frame**

![Final Frame Samples][final_frame]

_Notice how the pipeline performs well and is robust against shade from trees and bumps in the road._

---
_**Rubric Requirement**: Provide a link to your final video output. Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!)_
### Video Output
A video of the final output may be found [here](https://youtu.be/PTBO8joSabI)

---

### Optional Challenge
The optional challenge includes a video in which there are cracks in the road, lanes under bridges and contrasting pavement where roads have been repaved or fixed. This creates many artifcats which may be mistaken as lanes in the current pipeline. It is important to have good lane control logic to account for such things.

The current pipeline requires further parameter tuning and sanity checking to work on this video, but it provides good enough performance as a starting point for improvement.

---
_**Rubric Requirement**: Briefly discuss any problems / issues you faced in your implementation of this project. Where will your pipeline likely fail? What could you do to make it more robust?_
## Potential Shortcomings
1. It is not certain how well the lane detection will work when switching between lanes;
2. Roads passing under bridges and where there have been fixes may pose a problem to the lane filtering;
3. Winding roads may pose a problem to the current lane fitting, which is only second degree (quadratic);
4. White or yellow cars merging into the same lane may affect lane detection;
5. The performance of the current pipeline is not nearly close to what is required for real-time performance. Therefore, it may not be useful in real-world scenarios, unless GPU processing is used or it is coded in a faster language.

## Recommendations
Improvements to the pipeline should be made to address all the [potential shortcomings](#potential-shortcomings). The following are some potential starting points:
1. Improve the lane control logic to account for artifacts in the road by applying appropriate sanity checks to check the lane structure (such as minimum width);
2. Use a third-degree polynomial fit for lane fitting to account for winding roads.
3. Explore other ways of detecting lane lines, such as using a trained convolutional neural network, or generative adversarial network to predict the lane lines from a given frame.
