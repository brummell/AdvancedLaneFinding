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

[chessboard_distortion_correction]:  https://github.com/brummell/AdvancedLaneFinding/blob/master/chessboarddistortioncomparioson.png?raw=true "Chessboard Distortion Correction"
[car_distortion]: https://github.com/brummell/AdvancedLaneFinding/blob/master/cardistortioncomparioson.png?raw=true "Chessboard Distortion Correction"
[hsv_threshold]: ./hsv_thresholds "HSV Thresholding Examples"
[sobel_insufficieny]: ./sobel_examples.png "Sobel Thresholding Examples"
[histogrammed_lanes]: ./histoexample.png "Histogram Method for Lane-Finding"
[curvature_equation]: ./curvature_eq
[pipeline_example]: ./samplepipeline.png "Output"
[video1]: ./project_video.mp4 "Video"

###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

*see cell 5 of ipython notebook

Using the undistorted and distorted chessboard calibration images provided by the repository (chessboards are common due to their high contrast, regular structure, and wide availability, but their are other forms used for this purpose, including circles) to create a set of 3d "object" points, i.e. a set of points representing the true position of objects in space (but, in our case, at an arbitrarily defined 0 for all z coordinates) as well as a set of image points, which are points describing the same objects as represented in our images (and thus are 2d. Using the points from 17 of the 20 provided images (the openCV convenience method "findChessboardCorners" failed on two of the inputs, we can find the distortion coefficients and the camera matrix that will allow us to, generally, project our distorted images onto the correct space, i.e. without radial or translational distorition.

See below for an example of using our results to correct one of the chessboard images:

![alt text][chessboard_distortion_correction]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.

Using the distortion correction method, with my previous calculated coefficients of distortion and camera matrix, we can easily correct the distortions for a picture captured by this camera/lens. The distortion correction is fairly minimal, but the sign in the middle left and shrubbery in the middle right reveal that the images radial distortion has been corrected. See below:

![alt text][car_distortion]


####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

*see cell 5 of ipython notebook

I spent quite a bit of time working on this aspect of the project, and found it quite interesting in terms of the implied math. 

The first thresholds I worked were all based off of the Sobel operator, which is a --traditionally- 3x3 operator, the product of a differentiation kernel and an averaging kernel. Thus, when applied against the greyscale image intensities, it yields smoothed gradients of those intensities. The first two I tried we the individual x and y component gradients. These were somewhat useful individually, but it turns out that the best use of them was to create a filter from their logical conjunction, as the y gradient alone tended to pick up gradients in places as treelines, etc., all of which were useless. Thus the conjunction allowed the component gradients to reinforce eachother as well as threshold certain gradients unlikely to be useful for finding lane lines. 

The next pair of thresholds I used were based on the overall gradient, specifically the magnitude of the total gradient (found by taking the L^2 norm of the components) and the direction of the total gradient (found by taking the arctan of their y/x division). In trying to reassure myself of the correctness of the code, I spent some time convincing myself that lane lines are such that using the absolute value of the component gradients to find the direction in a reduced space (pi/2) did in fact allow one set of thresholds to capture all of the approrpriate angles, which was a fun reminded of high school geometry. That said, I didn't find much additional gain from these thresholds; parameterizing them was frustratingly empirical and they they often struggled to pick out the gradient of the yellow lane against the brightly lit street (though, I found HSV sufficient, and didn't have to, moving to single channel probably could've corrected this). 

In fact, even in a number of logical combinations, the Sobel masks were at best, acceptable performers, at least compared to the star of my whole pipeline, HSV thresholding.
See below: 

![alt text][sobel_insufficieny]


HSV, a cylindrical mapping of RGB, is a common color space for use in computer vision. It paints images as three layers: hue, saturation, and value. Using this system, and a helper method I wrote for my first lane finding project, I found it much easier to define the color thresholds I was interested in. In particular, I found a lot of success using an online tool that yields HSV coordinates from the pixel on an uploaded image to determine how the distant yellow lines differed from the sunlit street in the example pictures. This was an extremely useful method, and one that I will be sure to use in any similar endeavor. With my yellow and white spectrum thresholds, I simply made another mask from their conjunction and applied it to the image (converted to HSV, of course). This method pulled the lanes out of every sample picture with very little parameter adjustment. Granted, this is still an empirical method, and limited (at least in my implementation) to lanes of white and yellow, under reasonable light (shadow was not an issue though), but I think for this use case, it is probably an acceptable sacrafice, as I can think of no other common lane colors in this country. 


I should also note, I masked out an entire triangle-negative region in my images. This cut down tremendously on noise in the later steps, and can be made sufficiently loose as to ensure lane inclusion on all highway driving with this camera, regardless of the incline of the road. 

![alt text][hsv_threshold]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

*see cell 5 of ipython notebook

To obtain the required "birds-eye" perspective of the lane, I created a perspective transform that would allow me to shift the apparent z-coordinates in my image to produce an image with more accurately sized lanes, thus easing the intensity histogram method below (both by augmenting the relevant pixel count and regularizing the geometry of the pair of lines, making it easier to "cheat" at the initial guess for the lanes in each subsequent window (again, see that method's notes for further detail). 

Unfortunately, this was again a highly hands-on, if less obscure process. In order to create the transform, we require four points (definining a quadrilateral) on an original image, as well as the four points defining the shape and size of the desired quadrilateral (this is again accomplished through the creation of a matrix with homogeonous coordinates used to map through a 3d space and into an alternative projection). I felt the simplest way to do this was to take the lane lines from a straight lane, select the appropriate points from the trapezoid it "bounded" and then provide an appropriately sized rectangle of coordinates to map it onto. This was not hard to do, but again, it was very much a manual process. As suggested, I applied my transform to a number of images to ensure that it projected into an appropriate form (e.g. lane lines being mostly parallel, most of the distortion being well away from the road). See below:
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

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

*see cell 5 of ipython notebook

In order to find lanes on my manipulated image, I modified the Udacity code that takes a half-height, bottom slice of the thresholded binary image, creates a histogram of pixels, columnwise, and then extracts the maximum to set the anticipated center at the baseline. Then, for n windows, we begin to take 1/n slices of the image, and enclose rectangles of specified size around the previously generated x value. Within this rectangle, we record all of the hot, or 1-valued points, if they exceed an empirically set value, we find the centroid of these points and use it to define a new expected mean x value for the lanes in the subsequent 1/nth slice of the image. See image below

![alt text][histogrammed_lanes]

Following the gathering of these hot points, we simply use numpy's polyfit method to fit a polynomial (in this case, of degree, but one could imagine a system where a third degree may be necessary, assuming a sufficiently winding, highly visible road) for each set of points (left and right). 

I had worried that the histogram may wind up with insufficient points to correctly track highly curved lanes, but I found that the base implementation worked out-of-the-box with my HSV thresholding on all images and and every frame of the video, and so for the sake of time, left it as is. There are a number of possible solutions to this matter, including a reset after a number of failed windows to a full width histogram, possibly deeper as well, as the rough location of the loss-of-track, that would use all available information (at the cost of efficiency) to reground the algorithm. This could also be augmented by extrapolation of the found polynomials of the previous successful slide, or even the radius of curvature (described below). 


####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

*see cell 5 of ipython notebook

Radius of curvature, corresponding geometrically to the radius of the circle defining the curve, and algebraically, to a simple equation of first and second derivatives of the curve (for a graph). See below

![alt text][curvature_equation]

To find the radius I chose the y-value 1/3 of the way from the bottom of the warped image (top of the axis), this seemed a reasonably low-distortion, useful position for the curvature calculation). Entering this into the equation yielded RoCs for each lane line. Unfortunately, these were in terms of pixels. Fortunately, the code provided the means to convert pixels to x and y distances in meters (done empirically) and to then refit the polynomials and then once again find the RoCs. 

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I combined all necessary functions into a single, but simple processing pipeline which could be passed an image, either from an image file or a video. An example of the results using a still is presented below.

![alt text][pipeline_example]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
