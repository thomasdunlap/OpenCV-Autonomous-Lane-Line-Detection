# **Advanced Lane Finding Project**
![Image from final video][vid_img]
----
## The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[undist]: ./output_images/undist.png "Undistorted"
[undist_road]: ./output_images/undist_road.png "Road Transformed"
[orig_sobel_x]: ./output_images/orig_sobel_x.png "Sobel X-axis Transformation"
[hls_thresh]: ./output_images/orig_hls_thresh.png "Histogram Threshold Image"
[final_thresh]: ./output_images/final_thresh.png "Final Threshold Example"
[grad_dir_comb_thresh]: ./output_images/grad_dir_comb_thresh.png "Gradient Directions"
[perspective_wlines]: ./output_images/perspective_wlines.png "Lines compare perspective."
[sobel_y_thres_mag]: ./output_images/sobel_y_thres_mag.png "Sobel Y-axis and  Magnitude Images"
[warped_mask]: ./output_images/warped_mask.png "Warp Example"
[final_warped]: ./output_images/final_warped.png "Final Threshold and Warped Images"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[vid_img]: ./output_images/video_img.png "Output"
[video1]: ./project_video.mp4 "Video"


----

### Writeup / README

#### 1. Provide a README that covers all the [rubric](https://review.udacity.com/#!/rubrics/571/view) points and how you addressed each one.   

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook [Advanced-Lane-Lines.ipynb](https://github.com/thomasdunlap/CarND-Advanced-Lane-Lines/blob/master/Advanced-Lane-Lines.ipynb).  
I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the three-dimensional space of the real world. Here I am assuming the chessboard is fixed on the (x, y) plane at z = 0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![Distorted and undistorted checkerboard comparison.][undist]

## Pipeline

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Road image and undistorted road image.][undist_road]

I created an undistort function (IPython code block 4):

```python
def undistort(image, visualize=False):
    """
    Returns RGB numpy image, ret, mtx, and undistorted image, and can also do comparative visualization.
    """
    image = mpimg.imread(image)
    h, w = image.shape[:2]
    #objpoints, imgpoints = calibration_points()
    ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(objpoints, imgpoints, (w,h), None, None)

    # undistort
    dst = cv2.undistort(image, mtx, dist, None, mtx)

    if visualize:
        compare_images(image, dst, "Original Image", "Undistorted Image", grayscale=False)

    return image, mtx, dist, dst
```

#### 2. Describe how you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. Here is a comparison between the original image, and a Sobel x-axis transformed image:

![Original image and sobel x-axis transformed image.][orig_sobel_x]

 Sobel transformations basically return a binary image - if a specified number of gradients are all in the same direction forming a line, the function returns 1's (white) for those pixels, and 0's (black) if lines aren't found.  X-axis Sobel transformations work well for identifying vertical lines, while y-axis Sobel transformations are better at vertical lines.  I then took a maginitude threshold of the combined x- and y-axis transformations, where only a 1 was returned if both Sobels had 1's in that location, otherwise it returned a black zero. Here's an example of a y-transformation and magnitude threshold images:

![Sobel y-axis and magnitude thresholded images.][sobel_y_thres_mag]

Then I took a binary image of the gradient direction, and further combined that with the magnitude threshold image:

![Gradient direction and combined thresholds images.][grad_dir_comb_thresh]

Saturation is also a great way to identify lane lines.  I converted the RGB image to HLS (Hue, Lightness, Saturation), and then just kept a saturation-thresholded binary image:

![Thresholded image.][hls_thresh]

Finally, all of these thresholded, binary images were combined.  Here is the unidstorted orignal image compared to the final combined binaries image:

![Final combination of all thresholds.][final_thresh]

#### 3. Describe how you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in code cell 12.  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose to hardcode the source and destination points in the following manner:

```python
src = np.float32([[180, img.shape[0]], [575, 460], [705, 460], [1150, img.shape[0]]])

dst = np.float32([[320, img.shape[0]], [320, 0], [960, 0], [960, img.shape[0]]])
```

This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 575, 460      | 320, 0        |
| 180, 720      | 320, 720      |
| 1150, 720     | 960, 720      |
| 705, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image:

![Original and warped with lines images.][perspective_wlines]

I also created a mask to block out points outside the lane lines:

![perspective transform image with mask][warped_mask]

#### 4. Describe how you identified lane-line pixels and fit their positions with a polynomial?

I find lane-line pixels and fit their positions by taking the `warped` image and running it throught the function `detect_lines()` in IPython cell 13.

I split the image in two, and take the max value of the histogram of each side as the lane line points.

```python
def find_base_pts(warped):
    """
    Returns midpoint of image, and histogram peaks on either side of midpoint.
    """
    # Take a histogram of the bottom half of the masked image
    histogram = np.sum(warped[warped.shape[0] // 2:,:], axis=0)

    # Find the peak of the left and right halves of the histogram
    midpoint = np.int(histogram.shape[0] / 2)
    leftx_base = np.argmax(histogram[:midpoint])
    rightx_base = np.argmax(histogram[midpoint:]) + midpoint
    return midpoint, leftx_base, rightx_base
```

Then I used "sliding windows" to look for new histogram maxes:

```python
    # Number of sliding windows
    nwindows = 9
    # Height of windows: e.g. 720/9=80
    window_height = np.int(warped.shape[0] / nwindows)

    # Identify the x and y positions of all nonzero pixels in the image
    nonzero = warped.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])

    # Current positions to be updated for each window
    leftx_current = leftx_base
    rightx_current = rightx_base
    # Set the width of the windows +/- margin
    margin = 100
    # Set minimum number of pixels found to recenter window
    minpix = 50
    # Create empty lists to receive left and right lane pixel indices
    left_lane_inds = []
    right_lane_inds = []

    # Sliding windows
    if (left_line.detected == False) or (right_line.detected == False) :
        # Step through the windows one by one
        for window in range(nwindows):
            # Identify window boundaries in x and y (and right and left)
            win_y_low = warped.shape[0] - (window + 1) * window_height
            win_y_high = warped.shape[0] - window * window_height
            win_xleft_low = leftx_current - margin
            win_xleft_high = leftx_current + margin
            win_xright_low = rightx_current - margin
            win_xright_high = rightx_current + margin

            # Draw the windows on the visualization image
            cv2.rectangle(out_img, (win_xleft_low, win_y_low), (win_xleft_high, win_y_high), (0,255,0), 5)
            cv2.rectangle(out_img, (win_xright_low, win_y_low), (win_xright_high, win_y_high), (0,255,0), 5)

            # Identify the nonzero pixels in x and y within the window
            good_left_inds = ((nonzeroy >= win_y_low) & (nonzeroy < win_y_high) &
                              (nonzerox >= win_xleft_low) & (nonzerox < win_xleft_high)).nonzero()[0]
            good_right_inds = ((nonzeroy >= win_y_low) & (nonzeroy < win_y_high) &
                               (nonzerox >= win_xright_low) & (nonzerox < win_xright_high)).nonzero()[0]

            # Append these indices to the lists
            left_lane_inds.append(good_left_inds)
            right_lane_inds.append(good_right_inds)

            # If you found > minpix pixels, recenter next window on their mean position
            if len(good_left_inds) > minpix:
                leftx_current = np.int(np.mean(nonzerox[good_left_inds]))
            if len(good_right_inds) > minpix:        
                rightx_current = np.int(np.mean(nonzerox[good_right_inds]))

        # Concatenate the arrays of indices
        left_lane_inds = np.concatenate(left_lane_inds)
        right_lane_inds = np.concatenate(right_lane_inds)
        left_line.detected = True
        right_line.detected = True
    else:
        left_lane_inds = ((nonzerox > (left_line.current_fit[0] * (nonzeroy**2) +
                                       left_line.current_fit[1] * nonzeroy +
                                       left_line.current_fit[2] - margin)) &
                          (nonzerox < (left_line.current_fit[0] * (nonzeroy**2) +
                                       left_line.current_fit[1] * nonzeroy +
                                       left_line.current_fit[2] + margin)))
        right_lane_inds = ((nonzerox > (right_line.current_fit[0] * (nonzeroy**2) +
                                        right_line.current_fit[1] * nonzeroy +
                                        right_line.current_fit[2] - margin)) &
                           (nonzerox < (right_line.current_fit[0] * (nonzeroy**2) +
                                        right_line.current_fit[1] * nonzeroy +
                                        right_line.current_fit[2] + margin)))
```
Finally we use the extracted left and right line pixel coordinates in the `np.polyfit()` function to find the second order polynomials, and their respective x and y values:

```python
    # Extract left and right line pixel positions
    leftx = nonzerox[left_lane_inds]
    lefty = nonzeroy[left_lane_inds]
    rightx = nonzerox[right_lane_inds]
    righty = nonzeroy[right_lane_inds]

    # Here we need to save successful fit of lines to prevent case with empty x, y
    if (len(leftx) < 1500):
        leftx = left_line.allx
        lefty = left_line.ally
        left_line.detected = False
    else:
        left_line.allx = leftx
        left_line.ally = lefty
    if (len(rightx) < 1500):
        rightx = right_line.allx
        righty = right_line.ally
        right_line.detected = False
    else:
        right_line.allx = rightx
        right_line.ally = righty

    # Fit a second order polynomial to each
    right_fit = np.polyfit(righty, rightx, 2)
    left_fit = np.polyfit(lefty, leftx, 2)

```


#### 5. Describe how you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I found the position of the vehicle relative to center with the function `offset()`, and the radius of curavture with `curvature_with_offset()`, which are both located in code block 14.

`offset()` begins by finding the bottom-most (x, y) coordinates of the left and right lanes, and averages those points together to calculate the lanes midpoint:

```python
    # Calculate x bottom position for y for left lane
    left_lane_bottom = (left_fit_cr[0] * (y_eval * ym_per_pix) ** 2 +
                        left_fit_cr[1] * (y_eval * ym_per_pix) + left_fit_cr[2])
    # Calculate x bottom position for y for right lane
    right_lane_bottom = (right_fit_cr[0] * (y_eval * ym_per_pix) ** 2 +
                         right_fit_cr[1] * (y_eval * ym_per_pix) + right_fit_cr[2])
    # Calculate the mid point of the lane
    lane_midpoint = float(right_lane_bottom + left_lane_bottom) / 2
```

Then it calculates the midpoint of the image in meters:

```python  
    image_mid_point_in_meter = 1280/2 * xm_per_pix
    # Positive value indicates vehicle on the right side of lane center, else on the left.
    lane_deviation = (image_mid_point_in_meter - lane_midpoint)
```


```python
# Define y-value where we want radius of curvature
    # I'll choose the maximum y-value, corresponding to the bottom of the image
    y_eval = np.max(ploty)
    left_curverad = (((1 + (2 * left_fit[0] * y_eval + left_fit[1])**2)**1.5) /
                     np.absolute(2 * left_fit[0]))
    right_curverad = (((1 + (2 * right_fit[0] * y_eval + right_fit[1])**2)**1.5) /
                      np.absolute(2 * right_fit[0]))

    # Define conversions in x and y from pixels space to meters
    ym_per_pix = 30 / 720   # meters per pixel in y dimension
    xm_per_pix = 3.7 / 700  # meters per pixel in x dimension

    # Fit a second order polynomial to each
    left_fit_cr = np.polyfit(lefty * ym_per_pix, leftx * xm_per_pix, 2)
    right_fit_cr = np.polyfit(righty * ym_per_pix, rightx * xm_per_pix, 2)

    # Calculate the new radius of curvature in meters
    left_curverad = ((1 + (2 * left_fit_cr[0] * y_eval * ym_per_pix +
                           left_fit_cr[1])**2)**1.5) / np.absolute(2 * left_fit_cr[0])
    right_curverad = ((1 + (2 * right_fit_cr[0] * y_eval * ym_per_pix +
                            right_fit_cr[1])**2)**1.5) / np.absolute(2 * right_fit_cr[0])
```
![alt text][image5]

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code block 15 in the function `map_lane()`. Here is the main chunk of that block that draws the lane lines:

```python
    # Create an image to draw the lines on
    warp_zero = np.zeros_like(warped).astype(np.uint8)
    color_warp = np.dstack((warped, warped, warped))

    # Recast the x and y points into usable format for cv2.fillPoly()
    pts_left = np.array([np.transpose(np.vstack([left_fitx, ploty]))])
    pts_right = np.array([np.flipud(np.transpose(np.vstack([right_fitx, ploty])))])
    pts = np.hstack((pts_left, pts_right))

    # Draw the lane onto the warped blank image
    cv2.fillPoly(color_warp, np.int_([pts]), (0,255, 0))
    # Warp the blank back to original image space using inverse perspective matrix (Minv)
    new_warp = perspective_transform(color_warp, inv=True)
    # Combine the result with the original image
    weighted_img = cv2.addWeighted(undistorted, 1, new_warp, 0.3, 0)

```

This function starts by creating `warp_zero` a blank image to draw lines on, and `color_warp`, which takes the `warped` grayscale image, and stacks in RGB layers. Next it creates `pts`, a horizontal stack of `pts_left` and `pts_right`, which hold our lane line drawing points.  The lane lines are then draw onto `color_warp` with `cv2.fillPoly()`, and the unwarped into `new_warp`.  `new_warp` now exists in the "real world" space, and is overlayed onto our undistorted image using `cv2.addWeighted()`.

I've additionally added a thumbnail video of `out_img`'s as the car moves forward in the video, and text displaying radius of curvature and vehicle offset in the rest of the code.  Here is an example of my result on a test image:

![Image of final video.][vid_img]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My pipeline becomes less stable when going around sharp curves, or in areas where there are a lot of shadows, or inconsistent lighting that the camera has to adjust to.  I'm sure the pipeline would also have trouble in snow or rain as well, or if there was construction  with a lot line confusing lines, or complete lack of lines.  It would also probably have a hard time in a city like Boston, where the roads can be difficult and confusing for even humans. We mostly have ideal conditions with the video, and you don't have to live long to know the world is rarely provides the conditions we expect.

Improvements could definitely be made by having a higher-quality camera that adjusts rapidly to light changes, possibly introducing precipitation-like noise to the image, and more testing under various conditions.  
