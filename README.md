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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The example codes already gives me the code to find corners and add images points and object points. I used `cv2.calibrateCamera(objpoints, imgpoints, gray.shape[::-1], None, None)` to calculate the distortion parameters. I encapsulate the code in a function and return the results as global variable which can be used anywhere

**The function only needs to run once**


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Then I used `cv2.undistort(img, MTX, DIST, None, MTX)` to correct the distortion, the results is below:

![undistortion](/output_image/camera_calibration.png)


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. I defined 5 functions to do this job:

**abs_sobel_thresh**
This function applies threshold to absolute value of either sobelx or sobely

**mag_thresh**
applies threshold to the magnitude of sobelx and sobely value

**dir_thresh**
applies threshold to the absolute value of the gradient direction arctan(sobely/sobelx)

**hls_select**
applies threshold to the S channel of hls color space

**combined_thresh**
combines the results of all selection methods into a final binary value:
* sobelx
* sobely
* magnitude
* direction
* hls space

The result is
![binary](/output_image/binary_image.png)

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I took these steps to get the source and destination points and get a transformed image:
* Apply a distortion correction to raw images.
* Define the source and distination by manually select from the printed image
* Given src and dst points, calculate the perspective transform matrix
* Warp the image using OpenCV warpPerspective()
* Print the original image, and the warped image to verify if the bird eye view
* Print dots on the src corners

```python
src = np.float32([
        [590,450],
        [690,450],
        [1070,700],
        [230,700]
    ])
dst = np.float32([
    [300,10],
    [1050,10],
    [1050,700],
    [300,700]
])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 590, 450      | 320, 10        | 
| 230, 700      | 320, 720      |
| 1070, 700     | 1050, 720      |
| 690, 450      | 1050, 10        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text](/output_image/bird-view.png)

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

First of all I define Line class to sanitize and smooth the lines based recent frames:
* check if there are line detected
* Calculate the fit coefficients of the new x and y
* Check if it's approximately parallel to best fit 
* Check if recorded over n iterations
* remove the first collection of x
* Caculate the best fit of current recorded frames
* Reset the 
* Record this line as missing and check if it ecceeds the tolerance   

I wrote a functin **find_lines()** to take in an image and returns left fit and right fit. The steps are:
* Import global_left_line and global_right_line as global variables 
* Get a map of x,y
* If both line are detected, use look ahead search based on last frame
* Otherwise use sliding window search to 
* Extract left and right line pixel positions
* Add two lines to the Lines

```python
def find_lines(binary_warped):

    global global_left_line
    global global_right_line

    nonzero = binary_warped.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])

    if global_left_line.detected == True and global_right_line.detected == True:

        left_lane_inds, right_lane_inds = look_ahead_search(binary_warped, global_left_line.best_fit, global_right_line.best_fit)
    else:

        left_lane_inds,right_lane_inds = sliding_window_search(binary_warped, nwindows=9)
        

    leftx = nonzerox[left_lane_inds]
    lefty = nonzeroy[left_lane_inds] 
    rightx = nonzerox[right_lane_inds]
    righty = nonzeroy[right_lane_inds] 

    global_right_line.add(rightx, righty)
    global_left_line.add(leftx, lefty)
    
    return global_left_line.best_fit,global_right_line.best_fit
```

To test this function I used a test image and drew the lines returned from the function on the bird view:

![lines](/output_image/found-lines.png)

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calcualte curvature and position for each line `calculate_position` and `calculate_position` in the Line class 
Then in `process_image` and used the averge of left and right as the final value

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implement this in function `project_lanes()`
Here is an example of my result on a test image:

![projected](/output_image/plot-back.png)

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).
I wrote `process_image()` as the pipeline function.

Here's the pipeline function's output for all test images

![test_images](/output_image/test_images.png)

Here's the [link to my video result](./output_video/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

### Issues:
The major issue I got in this project was the lane mask jumps around in last 5 seconds when the car approaches the white section of the road. This was because of there are some frames that are hard to processa and detect the lane or the detected lines are way off the acutal lane, and the the look-ahead-search method that we used to detect the lane will get worse over time. 

To overcome this, I implement the Line class to record the last few frames and take average of the detected line points, so that the line will have better transition when the line changes and will have better bad frame tolerance, which result in smoother image. Also, I check if the detected line actually make sense, if either of the two lines is off, I use sliding-window method to detect lines just as the first time

### Possible failure:
I think the pipelie is very likely to fail in these situations:
* Very sharp turns, the ramp on the highway, for example.
* Multiple turns in short distance, like the ones we see in the harder challenge video
* Bad weather or low light condition
* Lane width changed on merging lanes

### Possible solutions:
* Use more complicated polynomial to represent lines for the snake turns situation 
* Dynamically select the source points when generate the bird-view image
* Multiple cameras in different angles for better transform result



