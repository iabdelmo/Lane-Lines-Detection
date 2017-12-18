
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

[image1]: ./output_images/undistort_output.png "Undistorted"
[image2]: ./output_images/undistort_output_TestImgExp.png "output of un-distortion stage on one of the test images"
[image3]: ./output_images/warping_output.png "warping stage output"
[image4]: ./output_images/combined_binary_output.png "gradient and coloring threshold output"
[image5]: ./output_images/sliding_window_search_output.png "sliding window search output"
[image6]: ./output_images/forward_search_output.png "forward search Output"
[image7]: ./output_images/curve_calc_ouput.png "curve calculation Output"
[image8]: ./output_images/lane_darwing_output.png "drawing lane Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd code cell of the project IPython notebook.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

I have built my Pipeline in a class called `LaneDetection`, The constructor of this class will take the camera distortion coefficients and camera matrix.

The user of this class will have only one public interface to use which is `ProcessImage()`, This function takes an image as input parameter and will run the whole pipeline chain over it.

The first step in this chain is "un-distorting the input image " using the camera parameters passed while creating the object and the "undistort" function from open CV then save the un-distroted image in class member variable called `img_undst`.

Here you can notice the correction by looking on the bottom left of the image and how it is corrected

###### Note: the distortion correction can be found in the notebook under the part "2. Distortion Correction"

#### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The 2nd step in my chain is warping the un-distorted image, So I have implemented a private member function in my class called `__Warper()`, which appears in lines 192 through 217 in the 4th cell in project IPython.  The `__Warper()` function takes as inputs an image (`img`),And inside this function I have defined a hard coded values for the source and destination points in the following manner:

```python

src = np.float32([(590,450),(710,450), (280,680), (1100,680)])
        
dst = np.float32([(450,0),(img_size[0]-450,0),(450,img_size[1]),(img_size[0]-450,img_size[1])])
 
```

where w and h are the width and height of the img, The output of this stage will be also saved in member variable called `warped`, So the user of the my class can easily retrieve the output of this stage by printing the value of this member var.

I verified that my perspective transform was working as expected by drawing the warped_img image against the img with the src points plotted on it:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The 3rd step in my chain is converting the warped image to "**Combined Binary**" image.
To achieve that I have used a combination of color and gradient thresholds to generate this combined binary image.

For the coloring thresholding, I have used two different color channels from two different color spaces; The **S-Channel** from **HLS space** and **R-Channel** from **RGB space**.
For each color channel I have created a binary image; S-binary image and R-binary image.

For the Gradient thresholding, I have here used 4-different gradient thresholding tech:

1.  Gradinet Sobel X Threshold
2.  Gradient Sobel Y Threshold
3.  Magnitued Threshold 
4.  Gradient Direction Threshold

Then I have created a combined gradient binary from the above 4 gradient binary images using the below combination:
```python
combined_grad_binary[((gradx == 1) & (grady == 1)) | ((gradmag == 1) & (graddir == 1))] = 1
```
Then I combined the grad binary image with S-binary image and the R-binary image using the below combination:
```python
 combined_binary[((s_binary == 1) & (combined_grad_binary == 1))| (r_binary == 1)] = 1
```
The code for this stage you can find it implemented in the class private member function which is called `__GradientColorThreshold()`.

The output of this stage:

![alt text][image4]

The 2nd column show the final **combined binary** output images from this stage and the 3rd show the **color binary** image that describes the contribution of the:

1.  Gradient combined binary -> Green color 
2.  R-binary -> Red color 
3.  S-binary -> Blue color

in creating the final combined binary image.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The 4th stage in the pipeline is identifying the lane lines pixels in X and Y positions.
In my pipeline I have 2 methods to identify the lane lines pixels:

1. Sliding window method that is implemented in private member function called `__SlidingWindowSearch()`
2. Forward search method that is implemented in private member function called `__ForwardSearch()`

The `ProcessImage()` function will choose which method to use according to the previous detection status; If the pipeline were correctly detetcted the lane lines on the previous image processing it will go through the forward search method else it will start a blind serach using the sliding window search method.

Both methods will apply 2nd order fitting on the identified lane lines pixels using `np.polyfit()` function.

The below graph shows the results for the sliding window search: 
![alt text][image5]

An example of the output of the forward search:
![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I used the same technique mentioned in the lectures by calculating the radius of curvature from the first and second derivative of the polynomial function and convert the result from pixel space to xy space. In the same method I calculated also the distance to center. The implementation can be found in private member function called `__CalcCurvature()`, I reused here many code parts from the lectures and from forums also to verify I'm calculating it correctly. I tested it on the following image and the results was (Radius of curvature for example: 22.293340952 m, 0.081 m Distance from lane center which looks reasonable ! 

###### Note the below image is output of private member function called `__DrawData()`

![alt text][image7]


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in private member function called `__DrawLane()` and this briefly uses MINV to inverse the Warping that happened in perspective transform and warp it back to the original image then overlays the resulted polygon to the original image and here is the output of the previous test image: 

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://github.com/iabdelmo/Lane-Lines-Detection/blob/master/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The main issue I have faced is: The behavior of my pipeline if the lane line detection failed. Should I invalidate all calculated data for the best fit polynomial and the current fit list or just drop this invalid detection data and start again using the window search and append its data on the previous results.

Actually I have used the following technique when the detection of lane lines fails; The pipeline will clear the current fit list and use the previous bestfit poly. in the lane drawing and radius curve calc. Also it works on the project video but I have a small flicker that I cannot resolve.

The improvement that I thought about it quickly I could make the perspective transform more robust (not with fix values) but dynamically calculated depending on the case. 
