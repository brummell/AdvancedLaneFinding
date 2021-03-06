# AdvancedLaneFinding
Advanced lane finding project for Udacity SDC program
---
The goals / steps of this project are the following:

* ~~Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.~~
* ~~Apply a distortion correction to raw images.~~
* ~~Use color transforms, gradients, etc., to create a thresholded binary image.~~
* ~~Apply a perspective transform to rectify binary image ("birds-eye view").~~
* ~~Detect lane pixels and fit to find the lane boundary.~~
* ~~Determine the curvature of the lane and vehicle position with respect to center.~~
* ~~Warp the detected lane boundaries back onto the original image.~~
* ~~Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.~~

[//]: # (Image References)

[chessboard_distortion_correction]:  ./chessboarddistortioncomparioson.png "Chessboard Distortion Correction"
[car_distortion]: ./cardistortioncomparioson.png "Chessboard Distortion Correction"
[hsv_threshold]: ./hsv_thresholds.png "HSV Thresholding Examples"
[sobel_insufficieny]: ./sobel_examples.png "Sobel Thresholding Examples"
[warp_points]: ./pointsforperspective.png "Point Selection for Perspective Warp"
[warped_car]: ./warped.png "Example of Image Warp"
[histogrammed_lanes]: ./histoexample.png "Histogram Method for Lane-Finding"
[curvature_equation]: https://wikimedia.org/api/rest_v1/media/math/render/svg/8c371c3a26f78a89d42d5a8b9b5e8a55d53600b8
[pipeline_example]: ./samplepipeline1.png "Pipeline Output"
[video1]: ./white.mp4 "Video Output"

###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

*see cell 3 of ipython notebook

Using the undistorted and distorted chessboard calibration images provided by the repository (chessboards are common due to their high contrast, regular structure, and wide availability, but there are other forms used for this purpose, including circles) to create a set of 3d "object" points, i.e. a set of points representing the true position of objects in space (but, in our case, at an arbitrarily defined 0 for all z coordinates) as well as a set of image points, which are points describing the same objects but represented in our images (and thus are 2d). Using the points from 17 of the 20 provided images (the openCV convenience method "findChessboardCorners" failed on two of the inputs), we can find the distortion coefficients and the camera matrix that will allow us to, generally, project our distorted images onto the correct space, i.e. without radial or translational distorition.

See below for an example of using our results to correct one of the chessboard images:

![alt text][chessboard_distortion_correction]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.

Using the distortion correction method, with my previous calculated coefficients of distortion and camera matrix, we can easily correct the distortions for a picture captured by this camera/lens. The distortion correction is fairly minimal, but the sign in the middle left and shrubbery in the middle right reveal that the images radial distortion has been corrected. See below:

![alt text][car_distortion]


####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

*see cell 8 of ipython notebook

I spent quite a bit of time working on this aspect of the project, and found it quite interesting in terms of the implied math. 

The first thresholds I worked were all based off of the Sobel operator, which is a --traditionally- 3x3 operator, the product of a differentiation kernel and an averaging kernel. Thus, when applied against the greyscale image intensities, it yields smoothed gradients of those intensities. The first two I tried were the individual x and y component gradients. These were somewhat useful individually, but it turns out that the best use of them was to create a filter from their logical conjunction, as the y gradient alone tended to pick up gradients in places such as treelines, etc., all of which were useless. Thus the conjunction allowed the component gradients to reinforce eachother as well as threshold certain gradients unlikely to be useful for finding lane lines. 

The next pair of thresholds I used were based on the overall gradient, specifically the magnitude of the total gradient (found by taking the L^2 norm of the components) and the direction of the total gradient (found by taking the arctan of their y/x division). In trying to reassure myself of the correctness of the code, I spent some time convincing myself that lane lines are such that using the absolute value of the component gradients to find the direction in a reduced space (pi/2) did in fact allow one set of thresholds to capture all of the approrpriate angles, which was a fun reminder of how long ago high school geometry actually was. That said, I didn't find much additional gain from these thresholds; parameterizing them was frustratingly empirical and they they often struggled to pick out the gradient of the yellow lane against the brightly lit street (though, I found HSV sufficient, and didn't have to, moving to single channel probably could've corrected this). 

In fact, even in a number of logical combinations, the Sobel masks were at best, acceptable performers, at least compared to the star of my whole pipeline, HSV thresholding.
See below: 

![alt text][sobel_insufficieny]


HSV, a cylindrical mapping of RGB, is a common color space for use in computer vision. It paints images as three layers: hue, saturation, and value. Using this system, and a helper method I wrote for my first lane finding project, I found it much easier to define the color thresholds I was interested in. In particular, I found a lot of success using an online tool that yields HSV coordinates from the pixel on an uploaded image to determine how the distant yellow lines differed from the sunlit street in the example pictures. This was an extremely useful method, and one that I will be sure to use in any similar endeavor. With my yellow and white spectrum thresholds, I simply made another mask from their conjunction and applied it to the image (converted to HSV, of course). This method pulled the lanes out of every sample picture with very little parameter adjustment. Granted, this is still an empirical method, and limited (at least in my implementation) to lanes of white and yellow, under reasonable light (shadow was not an issue though), but I think for this use case, it is probably an acceptable sacrafice, as I can think of no other common lane colors in this country. 

I should also note, I masked out an entire triangle-negative region in my images. This cut down tremendously on noise in the later steps, and can be made sufficiently loose as to ensure lane inclusion on all highway driving with this camera, regardless of the incline of the road. 

![alt text][hsv_threshold]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

*see cell 5 of ipython notebook

To obtain the required "birds-eye" perspective of the lane, I created a perspective transform that would allow me to shift the apparent z-coordinates in my image to produce an image with more accurately sized lanes, thus easing the way for the intensity histogram method below (both by augmenting the relevant pixel count and regularizing the geometry of the pair of lines, making it easier to "cheat" at the initial guess for the lanes in each subsequent window (again, see that method's notes for further detail). 

Unfortunately, this was again a highly hands-on, if less obscure process. In order to create the transform, we require four points (definining a quadrilateral) on an original image, as well as the four points defining the shape and size of the desired quadrilateral (this is again accomplished through the creation of a matrix with homogeonous coordinates used to map through a 3d space and into an alternative projection). I felt the simplest way to do this was to take the lane lines from a straight lane, select the appropriate points from the trapezoid it "bounded" and then provide an appropriately sized rectangle of coordinates to map it onto. This was not hard to do, but again, it was very much a manual process. As suggested, I applied my transform to a number of images to ensure that it projected into an appropriate form (e.g. lane lines being mostly parallel, most of the distortion being well away from the road). See below:

![alt text][warp_points]

![alt text][warped_car]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

*see cell 11 of ipython notebook

In order to find lanes on my manipulated image, I modified the Udacity code that takes a half-height, bottom slice of the thresholded binary image, creates a histogram of pixels, columnwise, and then extracts the maximum from each side of the image to set the anticipated center at the baseline. Then, for n windows, we begin to take 1/nth slices of the image, and enclose rectangles of specified size around the previously generated x value. Within this rectangle, we record all of the hot, or 1-valued points, if they exceed an empirically set value, we find the centroid of these points and use it to define a new expected mean x value for the lanes in the subsequent 1/nth slice of the image and continue moving our way upward. See image below

![alt text][histogrammed_lanes]

Following the gathering of these hot points, we simply use numpy's polyfit method to fit a polynomial (in this case, of degree 2 --but one could imagine a system where a third degree may be necessary, assuming a sufficiently winding, highly visible road) for each set of points (left and right). 

I had worried that the histogram may wind up with insufficient points to correctly track highly curved lanes, but I found that the base implementation worked out-of-the-box with my HSV thresholding on all images and and every frame of the video, and so for the sake of time, left it as is. There are a number of possible solutions to this matter, including a reset after a number of failed windows to a full width histogram, possibly deeper as well, as the rough location of the loss-of-track, that would use all available information (at the cost of efficiency) to reground the algorithm. This could also be augmented by extrapolation of the found polynomials of the previous successful slide, or even the radius of curvature (described below). 


####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

*see cell 11 of ipython notebook

Radius of curvature, corresponding geometrically to the radius of the circle defining the curve, and algebraically, to a simple equation of first and second derivatives of the curve (for a graph).See below

![alt text][curvature_equation]

To find the radius I chose the y-value at the bottom of the warped image where the car would be located. Entering this into the equation provided by Udacity yielded RoCs for each lane line. Unfortunately, these were in terms of pixels. Fortunately, the code provided the means and measurements to convert pixels to x and y distances in meters (done empirically) and to then refit the polynomials and then once again find the RoCs. I found the RoCs to be reasonable for curved roads, but highly variant over short periods of time.

Deviation from center was calculated by comparing the center of the image to the measured middle between the two lanes (left + right x positions, divided by 2).

I also added a low-pass filter for RoC calculations as well as the car deviation to deal with the high variance. 

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I combined all necessary functions into a single, but simple processing pipeline which could be passed an image, either from an image file or a video. An example of the results using a still is presented below.

![alt text][pipeline_example]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./white.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I found this project to be considerably more straightforward than the others, including the first lane-finding project. Largely, I suspect this is because I walked into it already having written my HSV yellow/white thresholding function, and only needing to tune it slightly. Granted the histogram method proved very functional, but many of my fellow students who I conversed with also modified or outright used the same solution and had considerable issues in shadows, etc.. As I mentioned, I wound up not even using the Sobel methods I wrote. 

Even more surprising was the smoothness of the found lanes even without the moving average smoothing I used in my first project. I think that really speaks to a different kind inherent usefulness in using the histogram method compared to Canny/Hough, at least in some cases. 

OpenCV masked over much of the mathematical troubles in converting to homogeonous coordinates, etc. for the camera calibration and perspective transforms. Although I found the math very interesting as I read more and more about it, it certainly isn't lost on me how much I benefitted from the availability of said library. 


Concerning likely failure points, you can see many of them in the attempts at the challenge video, and even to some extent parts of the main video when using Sobel: primarily, the presence of irrelevant parallel lines such as barriers or roadwork artifacts throw the algorithm into a tizzy. I suspect this could be corrected in a not-to-algorithmically way in my chosen system, but it would require even additional empirical parameters to choose only consistent lines that are a certain distance apart. I imaging the situation only getting worse in middle-lane driving. Moreover, see my comments in the poly-fitting section for my thoughts on the possible failure states of the histogram method and some imagineable solutions to those issues. 

Overwhelmingly, though, I'm reminded of one of my first attempts at the behavioral cloning project, in which my car drove off of the road into the parking lot, but despite the lack of road or markers, had apparently learned how to navigate by running parallel to large, warped rectangular paths. This was very early on in the project, and even still, my convnet showed a level of robustness I can't even imagine this method maintaining. The community, both amongst my classmates, and in the broader SDC community seems to already be heading this way for the same problem description.  

