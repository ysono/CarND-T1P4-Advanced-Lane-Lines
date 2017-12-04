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

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Submission

The code is divided into two jupyter notebooks, both of which are printed as html.

- The main [project.ipynb](project.ipynb), [printed](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html).
- A separate [project_threshold_analysis.ipynb](project_threshold_analysis.ipynb) used to optimize thresholds, [printed](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_threshold_analysis.ipynb.html).

Other files are:

- This [writeup](writeup.md).
- Video outputs [project_video_annotated-best.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_video_annotated-best.mp4), [project_video_annotated-final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_video_annotated-final.mp4), [challenge_video_annotated_final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/challenge_video_annotated_final.mp4), [harder_challenge_video_annotated-final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/harder_challenge_video_annotated-final.mp4).
- Additional test images were generated and saved in [test_images/](test_images), [challenge_video_test_images/](challenge_video_test_images), and [harder_challenge_video_test_images/](harder_challenge_video_test_images), grouped by camera calibration.
- Images used for explanation in this writeup are under [output_images/](output_images).
- Intermediary data saved by the notebooks are [calibration.p](calibration.p), [color_thresh_stats.p](color_thresh_stats.p), [abs_gradient_thresh_stats.p](abs_gradient_thresh_stats.p).

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is [here](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Calibrate-camera-using-chessboard-images).

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function.

Outputs of this operation can be found inside the notebook.

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Outputs of this operation can be found inside the notebook, in the same section as Camera Calibration above.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Thresholding was applied to each test image based on color and gradients, in one color space at a time.

Results were merged in the following fashion. For a given image:

- Threshold color value (0 to 255) in a few color spaces, and intersect the results across color spaces.
- Threshold gradient (0 to some value lower than 256) by their absoluste-value magnitudes in x and y directions, in a few color spaces, and intersect the results across color spaces.
- Threshold gradient by their L2-mean magnitudes from x and y directions, and by their angle, in a few color spaces, and intersect the results of magnitudes and directions across color spaces.
- Finally the above three results were merged using union (not intersection).

Because of the final union, an attempt was made to err on the side of keeping each of the three intersection results sparse (low recall) rather than dense (potentially low precision, i.e. many false positives). Therefore, it is not an issue that one method of thresholding yields near-blank results for some images.

Initially I attempted to use analysis of precision and recall to determine optimal thresholding. This required first capturing frames from the three provided videos and then manually defining desired solutions for left and right lines. In an iterative process, these frames were chosen where the most recent algorithm performed least well -- see examples in [output_images/badly_annotated/*](output_images/badly_annotated/). Precision and recall were helpful in eliminating obviously underperforming color spaces, but it did not tell the whole picture: even if two configurations yielded the same precision and recall, they could cover different pixels and thus be complementary. In the scheme of 20/80 rule of things, this was more effort spent than worthwhile.

In the end, the initial thresholds were generally chosen based on values mentioned in the lecture material, they were manually adjusted, their effects were drawn and visually evaluated. Adjustments were generally made on one paramter (e.g. reduce lower S-channel threshold, or increase Sobel kernel size) at a time, hence effectively performing a manual stochastic descent using the cost function of my qualitative evaluation. It's quite possible I missed the global optimum in each case.

The whole process of choosing and tuning thresholds can be read in this separate [project_threshold_analysis notebook](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_threshold_analysis.ipynb.html). A pattern emerge as 1) function `test_*_threshs` measures precision and recall 2) `analyze_thresh_stats` plots them 3) `*_thresh_test_images` fine-tunes thresholds. Optimized thresholds and algorithms were then copied into the [main notebook](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Detect-points-with-high-gradients-or-high-color-saturation) as functions `color_pipeline`, `abs_gradient_pipeline`, and `dirmag_gradient_pipeline`; and all of these were combined with function `line_detection_pipeline`. Outputs of this pipeline can be seen in this notebook [section](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Take-union-of-points-detected-by-gradient-and-by-color).

Some findings:

- I ended up using R,H,S channels for color thresholding and R,S channels for gradient thresholding. R channel was found to be useful for eliminating shadows and also for detecting yellow lines in `harder_challenge_video.mp4`.

- There was no need for a higher-end threshold except for H channel.

- Increasing kernel size smoothens out the result and increases captured points.

- Thresholding by direction alone produces sparse and noisy results. Applying intersection with L2-mean magnitude thresholding effectively filters in salient portions.

- The combination of direction and magnitude is not good at detecting small dots in the line (which in the real world are bumpy reflectors), and the use of absolute-value magnitudes were critical for this purpose.

- For L2-mean gradient thresholding, I attempted to weigh the squared x gradient heavier than the squared y gradient, but this was not found to make any real-life difference. See line `gradmag = np.sqrt((sobelx ** 2) * x_y_ratio + sobely ** 2)`.


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Code deriving perspective warping is situated before that for thresholding, because while tuning, warping was required to visualize the effect of thresholding configuration and estimate performance of the eventual line fitting. However, in the final pipeline for videos, warping is applied after thresholding is complete.

Warping involves the use of `cv2.getPerspectiveTransform` and `cv2.warpPerspective`. The notebook section is [here](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Warp-Perspective).

While optimizing threshold for frames captured from `challenge_video.mp4`, I noticed that the perspective in this video was different enough from that in `project_video.mp4` to warrant a different warping. For example, the diamonds marking HOV were too compressed. A tell-tale is that the bonnet looks different in the two videos. The final warping dimensions I chose are not perfect either but produce lines smooth enough to be fit by a 2nd-order polynomial (unless curvature changes quickly).

Frames from `harder_challenge_video.mp4` seem to have sufficiently similar perspective as those from `project_video.mp4` (they do have the same bonnets), but are warped separately anyway for another reason: the curvatures are very tight throughout, and reducing distance of vision helps eliminate noise. For the two challenge videos, top 500 pixels are ignored, vs top 450 in `project_video.mp4`.

Warping leaves small triangles of 0 weights at the bottom corners, but this does not affect line detection since warping is done after line detection.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code is [here](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Fit-lane-lines).

The pipeline works as follows:

- Find masks that most likely contain thresholded points that belong to a lane
- Collect those points within masks
- Use `np.polyfit` to fit a 2nd-order polynomial

As with the code from lecture, the initial (bottom) left and right centers are found by convolving a bottom portion of an image deeper than a single layer. Then, for each layer, we convolve within this layer, detect x coordinates with the highest convolution values, where the x coordinates are constrained using the previously found left and right centers with margins as guidance. This helps avoid capturing distracting wide shadows or other gradients, especially if they run parallel to a lane line. I find that the initial depth is sensitive in inducing wobbling of the distal tips and in mistaking noise for lines when curvature is acute.

The main differences between code provided by the lecture material and my code beyond refactoring are:

- The convolution window starts out at the bottom layer as 50-pixels wide, but is enlarged exponentially to 3x at 150-pixels wide at the top layer. The reason is that if any warped binary image contains excessively wide lines, using `np.argmax` on convolution output yields the smallest index, i.e. left-most window, resulting in a contribution of off-center points to polynomial fitting.
- The 'same' mode of convolution is used, to make the math of determining x coordinates from a convolution output easier.
- An attempt was made to use a window that contained negative weights in the periphery (and later collect points without using this periphery part), but this yielded equivalent or worse line detection, so this was scrappe. An attempt was also made to leave a hole in the middle, in hops it would detect double-lines better, but this was not successful either.

Attempted alternative window shapes:

```
  +1:       -------
   0:    ---       ---
-0.1: ---             ---
```

```
  +1:       --- ---
   0:    ---   -   ---
-0.1: ---             ---
```

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code over two contiguous sections [here](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#From-points-within-masks,-fit-curves,-and-calculate-vehicle-position) and [here](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#From-fit-curve,-find-radius).

NOTE, because I used different warping than provided by lecture material, the assumption of "30m per 720px and 3.7m per 700px" goes out the window. It is possible to calibrate the values using diamonds on the ground or US regulations for lengths of dotted lines and for widths of lines, but for simplicity, I recycled those two rates of conversion and applied them to all 3 videos.

The position of the vehicle is based on the fitted lines. It is based on the x-intercepts of the left and right lines and by how much their midpoint deviates from the midpoint of the frame.

```
vehicle_position = (x_lim / 2 - l_x_bottom) / (r_x_bottom - l_x_bottom) - 0.5
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

See contiguous sections [Draw](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Draw) and [Unwarp](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project.ipynb.html#Unwarp).

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

With the notebook run as-is, the output videos are

- [project_video_annotated-final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_video_annotated-final.mp4)
- [challenge_video_annotated_final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/challenge_video_annotated_final.mp4)
- [harder_challenge_video_annotated-final.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/harder_challenge_video_annotated-final.mp4)

In the course of development, I was able to produce a better video for `project_video.mp4`: [project_video_annotated-best.mp4](https://ysono.github.io/CarND-T1P4-Advanced-Lane-Lines/project_video_annotated-best.mp4). The threshold and algorithm that yielded this video are unfortunately lost, but they were discarded due to very poor performance on the other 2 videos. This "best" video is superior in that the tip of a left-curving dotted line does not wobble.

Note, the output contains undistorted frames, but the effects of camera calibration were not reversed.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Because we fit one line per left and right, a branching line would likely be badly detected. Also, while my radius estimation as-is is already very inacurate, existence of branches would further make them meaningless; in fact, relying on radius to make steering decisions could be very dangerous. To account for branching, a much more sophisticated algorithm is required, which probably takes in account the expected width of a lane and fine-tuned convolution window to allow dynamic nubmers of lines.

The top tip of the line is very senstivie to warping configuration. Gyroscope should be used to calibrate perspective.

Whereas the thresholding for gradient is based on percentage relative to maximum in the image (my code is written differently but implements the same logic as the code from lectures), I threshold absolute color values. Instead, we could use percentage relative to minimum and maximum, in both gradients and colors.

I did not use time averaging, as I did for lane finding in Project 1, but I would keep a history of approx 5 frames of the 2nd-order polynomial coefficients, reject coefficients from a new frame if they differ too much, and take a weighted average (favoring more recent frames).

The whole operation is slow and very unlikely to be real-time. Better hardware than my laptop or aws g2.2xlarge, and better lower-level libraries are required.
