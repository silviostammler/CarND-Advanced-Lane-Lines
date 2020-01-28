## Writeup Template

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undistort_output_calibration1.jpg "Undistorted"
[image2]: ./test_images/straight_lines2.jpg "Road Transformed"
[image3]: ./output_images/undistorted_straight_lines2.jpg "Undistorted test image"
[image4]: ./output_images/binary_combo_straight_lines2.jpg "Binary Example"
[image5]: ./output_images/warped_straight_lines2.jpg "Warp Example"
[image6]: ./test_images/test1.jpg "Original test image example"
[image7]: ./output_images/color_fit_lines_test1.jpg "Fit Visual"
[image8]: ./output_images/result_test1.jpg "Output"
[video1]: ./test_videos_output/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Helper functions

In general, all helper functions are contained in the fourth code cell of the IPython notebook "./P2.ipynb".

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the fourth (function `cal_undistort()` of Helper functions) and fifth code cell (pipeline for camera calibration and distortion correction, calculation of perspective and inverse perspective transfrom) of the IPython notebook located in "./P2.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a calibration image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the calibration image "./camera_cal/calibration1.jpg" using the `cv2.undistort()` function. The corresponding functionality is handled by the function `cal_undistort()`. I obtained this result: 

![alt text][image1]

The computed calibration matrix and distortion coefficients will be applied to each test image and frame in the video stream later on.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one ("./test_images/straight_lines2.jpg"):

![alt text][image2]

Using the calibration matrix and distortion coefficients as parameters in the function `cv2.undistort()` the undistorted image is generated ("./output_images/undistorted_straight_lines2.jpg"):

![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in fourth code cell `color_gradient_thresholding()`). The gradient thresholded image is built by combining a x gradient thresholded image, a y gradient thresholded image, a gradient magnitude thresholded image and a gradient direction thresholded image. On the other hand, the color transform refers to thresholding the S channel of the undistorted image. Here's an example of my output for this step ("./output_images/binary_combo_straight_lines2.jpg").

![alt text][image4]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `calc_perspective_trans_warp()`, which appears in the fourth code cell of the IPython notebook.  The `calc_perspective_trans_warp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  The source points were determined by looking at the undistorted thresholded binary image ("./output_images/binary_combo_straight_lines2.jpg"). To extract the source points region masking (`region_of_interest()`) and the Hough Transform approach (`hough_lines()`) is used to map out the full lane lines. Then, the four intersection points of the lane lines with the region of interest are the source points we are looking for (`calc_intersections()`). Instead the destination points are hardcoded.

```python
src = np.float32(
    np.array(intersection_vertices))
dst = np.float32(
    [[340,imshape[0]-1],
    [340,0],
    [940,0],
    [940,imshape[0]-1]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 217, 719      | 340, 719      | 
| 567, 470      | 340, 0        |
| 714, 470      | 940, 0        |
| 1108, 719     | 940, 719      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto the test image and its warped counterpart to verify that the lines appear parallel in the warped image ("./output_images/warped_straight_lines2.jpg"):

![alt text][image5]

The perspective transform matrix M and the inverse perspective transform matrix calculated based on these source and destination points will be utilized for any test image or frame in a video later on. Therefore, actual computation appears in the fifth code cell before the pipeline is executed on all of the test images.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The pipeline for identification of lane boundaries on test images is included in the sixth code cell.
Based on the warped image the function `find_lane_pixels()` computes the x and y coordinates of the left and right lane line pixels. This is done by utilizing a histogram for detection of base line positions as a starting point. Then, the image height is divided into a fixed number of sliding windows with a margin in x direction. Furthermore, the first window is centered at the base line position (at the bottom of the image). All of the pixels within a sliding window are identified as lane-line pixels. Depending on the mean x coordinate of detected pixels the sliding window is shifted to this position in the next step. This is done until all windows are covered (i.e. top of the image is reached). 
Taking the pixel positions of left and right lane-line as input the function `fit_polynomial()` fits a second order polynomial to the positions. For the original test image "./test_images/test1.jpg" ...

![alt text][image6]

the corresponding output ("./output_images/color_fit_lines_test1.jpg") is generated:

![alt text][image7]

By the way, the function `find_lane_pixels()` comprises another approach for finding the lane-line pixels besides the histogram-sliding window method. The histogram-sliding window approach is used when a confident solution hasn't been found yet or for the last n frames. On the other hand, there is a search from prior approach which calculates the lane-line pixels based on previous polynomial coefficients. This added funtionality will be important for finding the lane bounds in the video stream. 

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature is computed in the function `measure_curvature_real()`. Based on the x and y coordinates of the left and right lane-line pixels identified by `find_lane_pixels()` the left, right and average radius of curvature is calculated in meters. First, the polynomial coefficients has to be calculated in world space. The corresponding conversion parameters from pixel space to world space are contained in the second code cell. Now, the radius of curvature is evaluated for both the right and left lane line based on the maximum y coordinate (i.e. bottom of the image). The average radius of curvature is calculated as the arithmetic mean of left and right curvature.
The position of the vehicle with respect to center is computed in the function `measure_position_real()` in meters. Based on the polynomial lane line x coordinate at the bottom of the image the midpoint of the lane is calculated. Subtraction of the midpoint of the lane from the center of the image and afterwards conversion into world space results in the offset we are interested in.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the fourth code cell in the function `visualization_lane_bounds()`.  Here is an example of my result on the test image ("./test_images/test1.jpg"):

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

The pipeline for lane-line detection in a video stream is contained in the seventh code cell of the IPython notebook "./P2.ipynb". In general, it has the same structure as for processing the test images. However, it is extended by some functionality. In this context the class `Line()` has to be mentioned tracking all important parameters during lane-line detection (third code cell). Furthermore, in each iteration a sanity check is carried out testing whether the solution for the current frame is accepted. A look-ahead filter is also implemented (in function `find_lane_pixels()`) searching in the neighborhood of a previously detected solution. Finally, a reset mechanism handles the case when a maximum number of bad frames (`MAX_bad_frames`) in a row appears. The lane-line pixel detection is then switched back according to the search from scratch approach.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The gradient/color thresholds will certainly not be suitable for any scenario. For example, in the challenge_video my pipeline doesn't work optimal because there is pollution on the road making the detection harder to achieve. Furthermore, it would be nice to have adaptive thresholds depending on the surroundings(weather, lighting conditions).
Especially, in the harder_challenge_video my pipeline fails since there are very frequent and strong curves. In this case the polynomial solution must be determined from scratch more frequently. Moreover, the polynomial fits have to be extrapolated if the lane lines reach the boundaries of a frame. I also observed that passing road users are detected as lane lines which would be fatal!!!  
