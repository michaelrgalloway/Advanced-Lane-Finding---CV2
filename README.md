

---

#*Advanced Lane Finding Project*

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

[image1]: ./output_images/undistort_checkers.png "Undistorted"
[image2]: ./output_images/Original_Img.png "Road Transformed"
[image3]: ./output_images/mag_thresh.png "Binary Example"
[image7]: ./output_images/LUV_L_Channel.png "LUV"
[image8]: ./output_images/LAB_B_Channel.png "LAB"
[image9]: ./output_images/Binary_Combined.png "Combined"
[image4]: ./output_images/Warped_Poly.png "Warp Example"
[image5]: ./output_images/Binary_Warped.png "Fit Visual"
[image10]: ./output_images/bird_lines.png "Fit Visual 2"
[image6]: ./output_images/Result.png "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

In the ipython notebook included, under the 'Calibrate Camera', within the Get_Undistort_M function, I execute cv2.calibrateCamera to obtain the matrix transform required. 

Before this I feed the cv2 function findChessboardCorners a series of images to detect points
and I match these found points to a static predetermined grid that the points should be adjusted to. For each image we gain a new pair of chessboard corners and same object points. 

These lists are given to cv2.calibrateCamera() to obtain the matrices needed to undistort our
camera images. 

In another cell we define the function 'Undistort_Image' that will use the calibration that we just performed for every image in the proessing queue.

Here is an example of a before and after of checkerboard images.


![alt text][image1]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one. I chose this image below as a sample because this spot was the trickiest spot to obtain a well drawn space in the lines since the extra shadows and lines 
made this frame a little more difficult:
![alt text][image2]
####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
I experimented with several gradients and color spaces before decided to use my final binary image. Below is an example of a gradient magnitude threshold applied. 

![alt text][image3]

However the more I explored color spaces, I found I can extract line data better than the gradient threshold alone. I used LUV and LAB color spaces to achieve extracting line data. The LUV color space uses lightness intesity in L channel which we can use to obtain lighter areas of the image which we can determine as lines. I also used the LAB color space because in this space, the B channel is a spectrum of blue to yellow which we can use to extract out yellow and lighter white lines together with LUV. 

The LUV and LAB color thresholds applied look like this respectively

![alt text][image7]
![alt text][image8]

and added together as a binary image, the combined binary looks like this
![alt text][image9]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I created a function called 'def birdseye(img)' which applies a perspective transform on the given image with the hard-coded coordinate map show below. Return is the warped image, the
matrix that was used and also the inverse matrix so we have it later because we will need to
unwarp a new image we create later back into the original image. The warped image from this
function will have a birdseye perspective on the binary thresholded image we created earlier. 

```
def birdseye(img):
    src = np.float32([[595,450],[705,450],[1125,720],[200,720]])
    dst = np.float32([[325,0],[1100,0],[950,720],[325,720]])
    M = cv2.getPerspectiveTransform(src, dst)
    Minv = cv2.getPerspectiveTransform(dst, src)
    warped = cv2.warpPerspective(img, M,(1280,720), flags=cv2.INTER_LINEAR)
    return warped, M, Minv

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 325, 0        | 
| 705, 450      | 1100, 0      |
| 1125, 720     | 950, 720      |
| 200, 720      | 325, 720        |

Here is the original image compare with the warped image with our predicted lines also plotted on. The original image is of the vehicle turning lightly right, hence the curved lines. 

![alt text][image2]
![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I included in the code within the 'getNewFitCoefficientsAndPoints' method under the 'Lane Extraction and Impose' section of the jupyter notebook.

From the bottom of the warped binary image and working upward at a 50px interval, we determine the left and right peaks. We split the image into two sides horzontally and then from those two images we obtain a window towards the center of these two sides with a 100px horizontal window on each side. We then take the vertical sum of all columns in this window and grab the column with the highest sum (essentially taking the histogram of the window). 
 We then take these two sum maxes as our left and right peaks. Once we find out first peaks for thr bottom of the image, we can use these peaks as the starting point of repeating this process on our next block up the image. 
 
 After the entire image has been vertically traversed in this fashion, we will have a set of x,y coordinates for both left and right lines.
 With these coordinates we can obtain a polynomial using np.ployfit which will return us the polynomial  coefficients. We assign the x and y values of the current frame that we've found, along with the current polynomial that we've obtained from polyfit and we store them in our left_line and right_line global objects 

 If we already traversed the previous frame for the lines, searching the entire image for histogram peaks, then we can use the previous polynomial that we have obtained to take a better guess at which where the current line points will be. If this fails, if it happens that we cant find resonable lines based off the previous found polynomial, then we can start the search process over again that we started with in the beginning
 
 Here is the sample birdseye binary image that we are finding lines for

![alt text][image5]

And here is the image after fitting lines. (No Grayscale)

![alt text][image10]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculate the 2 curvatures (left and right) in 'getNewFitCoefficientsAndPoints' method where the comment 'Now to calculate curvatures' is. 

Using the formulas from the lessons, we used the meters per image pixel and augmented the found pixels in the images while looking for the lines. We then fit the augmented points with the numpy polyfit function. We then use this polynomial to calculate the curvature using the formulas in the lessons to do so. We do this for both left and right lines. 

We then average these to curvatures [ ( l + r ) / 2 ] at the bottom of the 'extractLanesAndImpose' method and print the result onto the image using cv2.putText().

In the same method, extractLanesAndImpose(), I obtian the position of the vehicle by calculating the predicted x points for both left and right lines for the value 720 (bottom of the image). We then take the mean of these two values to get the center value. To get the distance from the theoretical pyhisical center, we take the difference. With this difference we use the formula to convert pixel distance to meters. This is our deviation length from the center of the road. 

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Through the functions and operations composed in the jupyter notebook, the original image is annotated with the color filled between the detected lanes.

![alt text][image6]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_images/project_video_annotated.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Some challenges I faced during this project is constantly detecting outliers, I would run the image through my pipeline and I would see that wrong lines were sometimes being detected that shouldnt be. I eventually got around this by applying a region mask after warping the image. This limited the search space and ultimately the lane finding histograms constantly found the correct lanes. 

Another challenge was to research into alternative colr spaces that I was not yet familiar with. I had to do some extra research into HSL, HSV, LUV, LAB, and other color spaces that could maybe or maybe not solve the problem, however the extra gained exposure of various color spaces that I didnt know of before this project Im sure will prove valuable on other work in the future. Its very interesting and I will probably even research more into even without a direct application yet. 

If I were to expand upon this project to make it more robust, I would  try to make a more dynamic thresholding / color space system along with maybe dynamic region masking based on the orientation of the previous frame curve. I would like to look into other ways I can threshold the image data however make a more dynamic system that chooses different thresholds and color thresholds based on the previous one applied. I think a alot can be achieved if given time and R&D.

 

