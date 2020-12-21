# **Advanced Lane Lines** 

## Report by Reza Rajan

---

**Advanced Lane Lines**

The goals of this project are as follows:
* Calibrate a camera to account for its distortion properties
* Build a lane detection pipeline which finds lane lines on the road
* Fits lane lines to polynomial curves
* Finds the radius of curvature of the lane
* Calculates the vehicle offset from the road
* Reflect on the work in a written report


[//]: # (Image References)
[distorted]: ./report_files/distorted.jpg
[undistorted]: ./report_files/undistorted.jpg
[sobel]: ./report_files/sobel.jpg
[color_mask]: ./report_files/color_mask.jpg
[colgrad_mask]: ./report_files/colgrad_mask.jpg
[perspective_transform]: ./report_files/perspective_transform.jpg
[histogram]: ./report_files/histogram.jpg
[sliding_window]: ./report_files/sliding_window.jpg
[look_ahead]: ./report_files/look_ahead.jpg
[inverse_transform]: ./report_files/inverse_transform.jpg
[final_frame]: ./report_files/final_frame_multi_sample.jpg

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
This step is required prior to any lane finding, **since distortion effects of the camera will affect proper, accurate line estimates.**

The camera is calibrated using the chessboard method. In essence, multiple images of a chessboard pattern, taken from the same camera, are used to calculate the distortion coefficients and camera matrix. Once obtained, these are used to perform a perspective transform on each of the camera's images.

### Distortion Correction
With the distortion coefficients and camera matrix, the image is tranformed as shown below:

**Original Distorted Image**
![Original distorted image][distorted]

**Undistorted Image**
![Unistorted image][undistorted]

### Color/Gradient/Lighting Thresholding
On the premise that lane lines are either white or yellow, are mostly vertical, and should contrast with the rest of the road, then **color, gradient and lighting may be used to distinguish lane lines from the road.**

A binary image is created by extracting yellow and white hues within certain lighting ranges from the image, using the HLS colorspace. Furthermore, the Sobel filter in the x-direction is used to extract the lane lines as vertical lines should contrast with the road.

It should be noted that a problem with this method is that lighting variations caused by shadows, for e.g. when passing under a bridge, may cause inconsistency in the thresholding. To overcome this, each image is normalized using Contrast-Limited Adaptive Histogram Equalization (CLAHE). This method is not perfect, but it helps make the pipeline more robust.

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

### Perspective Transform
To derive metrics from the lane lines, a top-down view is required, similar to plotting lines on a graph. This is required for lane fitting and calculating the radius of curvature. Thus, using four points on the image, it is transformed as follows:

**Perspective Transform**
![Perspective Transform][perspective_transform]

### Histogram Peak Extraction
With the perspective transformed binary mask, a method called histogram peak extraction is used to identify the positions of the base of the lane lines. 

In essence, this method sums the number of white pixel for each column of the binary image. The two columns with the highest sums are taken as the base of the lane. Note that this method assumes that lane lines are, for the most part, vertical.

**Histogram Peak Extraction**
![Histogram Peak Extraction][histogram]

### Sliding Window Lane Search
There is now enough information to start the lane line search. The sliding window lane search divides the top-down binary image into vertically-aligned grids, i.e. windows, starting at the previously found base positions of the lane. Starting with the first window, once a set minimum number of pixels are found within the window bounds they are stored as found lane pixels, and an average is taken as the next base position for the lane search in the next window, continuing for all windows to the top of the image.

This can be though of as _shifting_ the search window along the lane. Below is an illustration of this method:

**Sliding Window Lane Search**
![Sliding Window Lane Search][sliding_window]

### Searching from Prior Lane Bounds
An alternative to the sliding window lane search is searching from prior lane bounds. This method searched within a margin of previously found lane bounds, either from the sliding window lane search or recursively, from this method. This is done as a _look-ahead filter_ to improve the pipeline's performance by limiting the search bounds, and skipping the histogram peak extraction.

An illustration of this method is shown below:

**Look-Ahead Filter**
![Look-Ahead Filter][look_ahead]

### Calculating Metrics
With the lane line pixels found, a polynomial is fit to each line, and the radius of curvature and vehicle offset are found. These metrics are stored in a `Line()` class for both the left and right lane lines. As a further abstraction, a `Lane()` class is used to hold these two `Line()` classes, and perform sanity checks and other logic for lane estimation.

### Inverse Perspective Transform
With the lane lines found and plotted, an inverse perspective transform is used, which is the inverse of the initial transformation matrix, to transform from a top-down view to a front-facing (original) view.

**Inverse Perspective Transform**
![Inverse Perspective Transform][inverse_transform]

### Video Pipeline
The lane line, along with the metrics from the `Lane()` class, are plotted onto each frame of the camera's video feed to produce the lane detection visualization.

Samples of the final frame results are shown below:

**Final Frame**
![Final Frame Samples][final_frame]

_Notice how the pipeline performs well and is robust against shade from trees and bumps in the road._

---


---

### Optional Challenge
The optional challenge includes a video in which there are trees at the sides of the road and bumps in the road (which is also a different, lighter color compared to the previous videos), as well as the car's hood in the frame. Without the color filter in the pre-processing step, parameter tuning, and threshold checking with K-type control, there would be many edges passing through the filter. As a result, lane detection would have been difficult and may have resulted in skewed lines. However, since the pipeline had been designed with robustness in mind, the lane detection works decently in this scenario.

The below video demonstrates the results of the optional challenge:

![line-challenge][line-challenge]

Most notably, there is a point in the video where the car drives over a bump, but the lane line does not deviate significantly. This is a result of the thresholds set in the [lane line generation step](#3-lane-line-generation). Furthermore, once the car is back on track it appears as though the line resumes its update. This showcases the robustness of the pipeline against edge cases.

---

### A Note on Code Structure
Each of the above steps have been implemented as functions which are chained to form the pipeline. Structuring the code this way make followings the pipeline flow easy to understand, and allows for quicker parameter tuning. Most important is the *draw_lines()* function, which includes most of the logic for lane detection.

---

## Potential Shortcomings
1. It is not certain how well the lane detection will work when transitioning between lanes, since the region mask is tailored to only a single lane;
2. Environment conditions such as rain or snow, or even when driving in the dark or through areas with bright lights, may have significant impacts on the lane detection with the current pipeline;
3. White or yellow cars merging into the lane may affect lane detection as irrelevant edges may make it through the filter;
4. Currently the smoothness of lane detection is premised on the starting frame generating an appropriate line. If lane detection is started in a driveway, this inital estimate of lane may cause all other lane estimates to fail, due to the thresholds set;
5. In general, performing lane detection in the previously defined manner is perhaps not useful in real-world scenarios, as there are many edge cases which may pass through the filters regardless of parameter tuning.


## Recommendations
Improvements to the pipeline should be made to address all the [potential shortcomings](#potential-shortcomings). The following are some potential starting points:
1. Create dynamic region masks which may be based on an estimate of the vehicle's pose, and which will filter for more relevant regions, especially when changing lanes;
2. Employ the use of other types of sensors which are effective against rain, snow and bright lights in the visible spectrum to better predict the location of lane lines, especially in poor weather conditions;
3. Including object tracking can be useful to filter out merging cars from the list of detected edges;
4. Include lane reset logic such that if an initial estimate is off and results in corresponding frames which jitter too much (instability), the initial estimate will be cleared and updated at a certain frame. This could be useful when moving from a driveway to a road with lanes, which would usually be the case as lane tracking is most likely to start when the vehicle is turned on;
5. Explore other ways of detecting lane lines, such as using a trained convolutional neural network, or generative adversarial network to predict the lane lines from a given frame.
