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

[image1]: ./output_images/writeup/calibration_output.png "Calibration Output"
[image2]: ./output_images/writeup/road_undistortion.png "Road Undistortion"
[image4]: ./output_images/writeup/road_transformation.png "Road Transformed"
[image3]: ./output_images/writeup/binary_combo.png "Binary Example"
[image5]: ./output_images/writeup/warped.png "Warp Example"
[image6]: ./output_images/writeup/histgram.png "Histgram"
[image7]: ./output_images/writeup/fit_window_search.png "Fit Visual"
[image8]: ./output_images/writeup/output_2.png "Output"
[video1]: ./output_images/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.
This document is the project writeup. You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd code cell of the IPython notebook located in "./advanced_lane_lines.ipynb".

I start by preparing "object points", which will be the 3D (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.
Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard `corners` in a test image with `cv2.findChessboardCorners()`.  `imgpoints` will be appended with the (x, y) pixel position of each of the `corners` in the image plane with each successful chessboard detection.  

For visualization, I draw the corners on the images with `cv2.drawChessboardCorners()`. The output images are saved in `./camera_cal/found/` folders.

There are 3 images, `calibaration1`, `calibration4` and `calibration5`, which are incomplete to figure out 9x6 corners out of the images. So they are not counted in the calibration.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.

Finally, in 3rd code cell, I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:
![alt text][image1]

The calibration result is calculated once but used many times. So I saved the result as pickle file into `./calibration_pickle.p` for further use.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
I create a function called `undistort_image()` to apply the undistortion to an image. This is in the 5th code cell. In the function, I load the previously saved pickle to get the calibration result and apply them to an input image with calling `cv2.undistort()`.
Below is one of the test images undistorted:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image (in 6th code cell).
First, I convert the image color space from RGB to HLS. I select S channel to create a color threshold mask. The Saturation channel (S) of the HSL color space is a good way to highlight the lane lines.
Also I use a sobel filter in the X direction to highlight the horizontal edges in a mask.
Then I combine the 2 masks with bitwise OR opertaion to get a combined thresholded image.
 Here's an example of my output for this step.  
![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in the 7th code cell. The `warp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points as follows:

| Source        | Destination   |
|:-------------:|:-------------:|
| 595, 450      | 320, 0        |
| 200, 720      | 320, 720      |
| 1125, 720     | 1000, 720      |
| 685, 450      | 1000, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.
![alt text][image4]

There is still an input flag `inverse` to choose whether the image is warped from `src` to `dst` or vice versa. The inversed warp is helpful for future use in this project.

And the warped result of the combined thresholded image:
![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Firstly, I create a histogram of the pixels of the lower half position of the generated binary thresholded image. The each peak in left half and right half is selected as starting point. The histogram process is in the 9th code cell and below is the histogram:
![alt text][image6]
I set a left bound and right bound when calculating histogram to prevent left/right seprate wall edges.

Then I apply sliding windows from the starting points from bottom to top, to figure out all pixels in the range of window. If more than 50 pixels are found in range of one window, then these pixels are thought as part of the lane lines. The mean X positions of these pixels will be the next window center.
However, too few pixels in a window will be treated as noise and discarded. Then the center of next window for searching should be the mean X position of all the captured pixels. This is to prevent the searching off the lane for several continous windows(especially for the dashed lane lines.).

Once I finished all the windows search, I fit a polynomial with `numpy.polyfit()` function with all captured pixels. The polynomial is quardratic and different coefficients for left line and right line as follows:
![alt text][image7]
This part is in the 10th code cell.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.
I have got the polynomials of left and right lines, then according to the [tutorial](http://www.intmath.com/applications-differentiation/8-radius-curvature.php), I can calculate the curvature radius. I use the mean value of left curvature radius and right curvature radius as the final result.
It is in the 11th code cell.

I also figure out the center point of the lane. I find the bottom points(with max y) for the 2 polynomials in the image, then find out the distance between the center of the lane from the center of the image. The negative value means offset to the left while the positive value means the offset to the right.
This part is in the 12th code cell.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Finally, I highlight the lane area in the outptu warped image. Then I inversely warp the warped image into the original images. And I put curvature radius and lane offset value on the frame.

To prevent the lane highlight changing sharply, I add a weight for the current line fitting coefficients with 0.2.
This part is done in 13th, 14th and 15th code cells.

Here is the final output:
![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_images/project_video.mp4).

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

* When I was doing sliding window search, I found sometime the line cannot be well fit due to lack of many pixels in the dashed line in the frame. Sometimes the sliding window get off the actual curvature with the mean X position of pixels in the previous window. So I instead used the mean X position of the total processed windows to get a better result.
I think to apply a better mask may also help in this situation.

* In the parts in the lane where color or brightness change dramatically, the lines may have some wobbles. I added a weight for current line fittings to reduce the effect. However, I think it needs more frame preprocessing and better combined masks to fix this kind of cases and make the line robust.

* It is better apply the simple lane line finding method for efficiency of processing.  
