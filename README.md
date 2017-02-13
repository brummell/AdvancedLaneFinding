# AdvancedLaneFinding
Advanced lane finding project for Udacity SDC program


##Writeup Template
###You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---
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

###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

*see cell 5 of ipython notebook

Using the undistorted and distorted chessboard calibration images provided by the repository (chessboards are common due to their high contrast, regular structure, and wide availability, but their are other forms used for this purpose, including circles) to create a set of 3d "object" points, i.e. a set of points representing the true position of objects in space (but, in our case, at an arbitrarily defined 0 for all z coordinates) as well as a set of image points, which are points describing the same objects as represented in our images (and thus are 2d. Using the points from 17 of the 20 provided images (the openCV convenience method "findChessboardCorners" failed on two of the inputs, we can find the distortion coefficients and the camera matrix that will allow us to, generally, project our distorted images onto the correct space, i.e. without radial or translational distorition.

See below for an example of using our results to correct one of the chessboard images:

![alt text][chessboard_distortion_correction]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]


####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

*see cell 5 of ipython notebook

I spent quite a bit of time working on this aspect of the project, and found it quite interesting in terms of the implied math. 

The first thresholds I worked were all based off of the Sobel operator, which is a --traditionally- 3x3 operator, the product of a differentiation kernel and an averaging kernel. Thus, when applied against the greyscale image intensities, it yields smoothed gradients of those intensities. The first two I tried we the individual x and y component gradients. These were somewhat useful individually, but it turns out that the best use of them was to create a filter from their logical conjunction, as the y gradient alone tended to pick up gradients in places as treelines, etc., all of which were useless. Thus the conjunction allowed the component gradients to reinforce eachother as well as threshold certain gradients unlikely to be useful for finding lane lines. 

The next pair of thresholds I used were based on the overall gradient, specifically the magnitude of the total gradient (found by taking the L^2 norm of the components) and the direction of the total gradient (found by taking the arctan of their y/x division). In trying to reassure myself of the correctness of the code, I spent some time convincing myself that lane lines are such that using the absolute value of the component gradients to find the direction in a reduced space (pi/2) did in fact allow one set of thresholds to capture all of the approrpriate angles, which was a fun reminded of high school geometry. That said, I didn't find much additional gain from these thresholds; parameterizing them was frustratingly empirical and they they struggled to pick out the gradient of the yellow lane against the brightly lit street. See below: 

![alt text][sobel_insufficieny]

In fact, even in a number of logical combinations, the Sobel masks were at best, acceptable performers, at least compared to the star of my whole pipeline, HSV thresholding.

HSV, a cylindrical mapping of RGB, is a common color space for use in computer vision. It paints images as three layers: hue, saturation, and value. Using this system, and a helper method I wrote for my first lane finding project, I found it much easier to define the color thresholds I was interested in. In particular, I found a lot of success using an online tool that yields HSV coordinates from the pixel on an uploaded image to determine how the distant yellow lines differed from the sunlit street in the example pictures. This was an extremely useful method, and one that I will be sure to use in any similar endeavor. With my yellow and white spectrum thresholds, I simply made another mask from their conjunction and applied it to the image (converted to HSV, of course). This method pulled the lanes out of every sample picture with very little parameter adjustment. Granted, this is still an empirical method, and limited (at least in my implementation) to lanes of white and yellow, under reasonable light (shadow was not an issue though), but I think for this use case, it is probably an acceptable sacrafice, as I can think of no other common lane colors in this country. 

![alt text][hsv_threshold]

I should also note, I masked out an entire triangle-negative region in my images. This cut down tremendously on noise in the later steps, and can be made sufficiently loose as to ensure lane inclusion on all highway driving with this camera, regardless of the incline of the road. 

![alt text][triangle_chop]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
