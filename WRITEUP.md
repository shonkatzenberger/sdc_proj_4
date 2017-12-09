# Advanced Lane Finding

Shon Katzenberger  
shon@katzenberger-family.com  
December 16, 2017  
October, 2017 class of SDCND term1

## Assignment (as specified in writeup_template.md)

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

[image0]: ./images/screen_shot.png "screen shot"
[image1]: ./images/hsv_v_pool_1.3.png "pool 1.3"
[image2]: ./images/unwarped_sobelx.png "sobelx"
[image3]: ./images/hsv_v.png "hsv v"
[image4]: ./images/hsv_s.png "hsv s"
[image5]: ./images/hls_s.png "hls s"
[image6]: ./images/sobel_on_challenge.png "sobel challenge"
[image7]: ./images/hsv_v_pool_tar.png "pool tar"
[image8]: ./images/pipeline_challenge.png "pipeline challenge"
[image9]: ./images/calibration2.jpg "calib2"
[image10]: ./images/calibration2_fixed.jpg "calib2 fixed"
[image11]: ./images/distorted_frame.jpg "distorted"
[image12]: ./images/undistorted_frame.jpg "undistorted"
[image13]: ./images/pipeline_mask.jpg "pipeline mask"
[image14]: ./images/persp_guides.png "perspective guides"
[image15]: ./images/rendered_lane_lines.png "rendered lane lines"
[image16]: ./images/overlay.png "overlay"
[image17]: ./images/pool_hsv_v.png "pool hsv-v"
[image18]: ./images/pool_hls_s.png "pool hls-s"

## Submission Details

The submission consists of the following files:
* `WRITEUP.md`: This file.
* `calibrate.py`: The script to compute and save the camera calibration values. This can be executed directly.
* `cameraCalibration.p`: The output pickle file from `calibrate.py`, containing the camera calibration values.
* `gui_show.py`: The visualization and pipeline harness application.
* `pipeline.py`: The computation pipeline code. This ***cannot*** be executed directly, but is imported by the harness, `gui_show.py`.
* `xforms.py`: Various visualization transforms, used for exploration, not by the final pipeline. This ***cannot*** be executed
directly, but is imported by the harness, `gui_show.py`.
* `videos/project.mp4`: Video of the pipeline executed on `project_video.mp4`.
* `videos/challenge.mp4`: Video of the pipeline (in sensitive mode) executed on `challenge_video.mp4`. This one contains some
minor glitches.
* `videos/harder.mp4`: Video of the pipeline (in sensitive mode) executed on `harder_challenge_video.mp4`. This one displays
major issues and is included 'just for laughs'. Getting the pipeline working on this video would require some major changes, particularly
to the `LaneLineInfo` class.
* `images`: This directory contains images referenced by this writeup.

Note that all code was written and executed under Windows 10, so may perform slightly differently on a different OS.

### Visualization and Pipline Harness Application

The file `gui_show.py` is my visualization and pipeline harness application. I highly encourage the reviewer to try it out.
It is a GUI application leveraging tkinter. I used this application for all experimentation, pipeline development, and generating
the final submission videos and the images for this writeup.

When executed, it prompts for a video file to load. To proceed, either select a video file, or cancel out of the open
dialog and press the "Load Pictures..." button to choose a directory containing images. For example, here is a screen shot
when the `test_images` directory has been loaded:

![alt text][image0]

* The first row of controls contains buttons to quit the application, load a video file, load a directory of images, save an image,
or save a video. Saving a video merits some additional explanation. Once the user has chosen an output file name, the application
resets to the first image in the dataset, then switches to "Run" mode, and saves each image as it is processed and displayed. The
submitted videos were generated this way. Note that the guide lines drawn by the "Draw guides" checkbox are ***not*** saved by either
"Save Image..." or "Save Video...", since they are not part of the image, but are extra objects on the main canvas.

* In the second row of controls, the "Run" checkbox is for rapidly scrolling through the images in the dataset. This is typically
faster than holding down the scroll bar arrow. The remaining checkboxes are for engaging the pipeline ("Use Pipeline"), whether to
overlay the computed lane line information on the original (undistorted) image ("Overlay Pipeline Result"), and whether to use the
"sensitive mode" of the pipeline ("Use Sensitive Settings"). See below for more detail on the pipeline. Note that the pipeline uses
Tensorflow, so the first time the "Use Pipeline" checkbox is checked, there is a significant lag before the canvas is updated, since
Tensorflow takes a while to spin up.

* The third row of controls is for experimentation. Most of these controls are overridden when "Use Pipeline" is checked. Here are some details:
  - Draw guides: Adds the perspective guide lines to the image canvas. Note that these are not part of the image, so are not saved by "Save Image..."
or "Save Video...".
  - Undistort: Corrects for the camera distortion.
  - Perspective Before: Applies perspective before the remaining controls are applied.
  - First dropdown: Various color space transformations and one "combo" transformation.
  - Second dropdown: The pooling and threshold settings. When this is anything but "none", we apply left and right one-sided mean pooling operations
(see below for more detail) of shape (11, 71), compute the max of these two pools, together with 20, then divide the pixel value by this max. This tends to
highlight local maxima. The indicated threshold is then applied (any value below the threshold will be set to zero). Note that this generally
makes sense only if "Perspective Before" is checked (otherwise, the pooling is a bit silly). Also note that this uses Tensorflow, so there will be a
lag the first time it is selected (to spin up Tensorflow). For example, with the `test_images` loaded, when "Undistort" is checked,
"Perspective Before" is checked, "hsv - v" is selected and "1.3" is selected, here is the result:

    ![alt text][image1]

  - Third dropdown: This is for specifying a derivative operation. If a multi-channel color option is specified (via the first dropdown) and pooling
is specified (via the second dropdown), then the "combo after LS pool" option also makes sense to apply. If you are curious, read the code for more
detail. This was for pure experimentation when developing the final pipeline. Note that this option may generate an AssertionError (displayed in the
console) if it is used when not appropriate.
  - Perspective After: This applies perspective after the other transforms. If "Perspective Before" is also specified, this undoes the perspective
transformation. This allows transforming in "warped" space, then visualizing in "non-warped" space. For example:

    ![alt text][image2]

* The "Item" box displays information about the current image, including the time required for transformations, measured in milliseconds.

* The scroll bar is for navigating through the set of images.

* The canvas displays the image, and guide lines, if specified. Note that if the canvas appears clipped by the application window, simply resize
the application window.

Note that this application can be used to open the final annotated videos (in the `videos` folder), so one can easily inspect each frame.
Of course, when one of these videos is loaded, the transformations should be turned off.

### Pipeline Details

The pipeline is implemented in the `getPipelineFunc` function in the `pipeline.py` file. The final pipeline I settled on consists
of the following operations:

* Correct for camera distortion.
* Apply the perspective transformation.
* Construct a two-channel image consisting of the V component of the HSV colorspace and the S component of the HLS colorspace.
The V component of HSV did a better job of highlighting both yellow and white lines than any of the other natural candidates,
namely grayscale, L of HLS, L of Lab, and L of Luv. The other component, S of HLS helps particularly when yellow lines are
on a bright background, for example, on the concrete bridges in `project_video.mp4`. For example, compare the following three images.
Notice how in the first, the yellow line blurs into the pavement, especially near the top of the image. In the second image, the yellow
stands out somewhat better, while the third is even better. The third also picks up the white lines better than the second.

  ![alt text][image3]
  ![alt text][image4]
  ![alt text][image5]

  The value of using S of HLS is even more apparent when the pooling threshold is invoked. Compare these images, the first using
  pooling and thresholding on V of HSV and the second using pooling and thresholding on S of HLS:

  ![alt text][image17]
  ![alt text][image18]

* From this step onward, I use Tensorflow for the computation. This step in the pipeline is the real magic. The goal is to find local
maxima of the channels. That is, we want to highlight pixels where the channel value is larger than the mean channel value across a wide
patch to either side (left or right) of the pixel. To do this, we compute the mean pool (with pool shape (11, 71)), then shift the result
each direction (by padding on each side) to approximate the one-sided mean pools. Finally, we divide the pixel values by the max of these
one-sided pools and a minimum threshold (20). A local maxima will be larger than both mean pools, so this quotient will be larger than one.
I found that this technique is superior to using derivatives, since derivatives highlight steep change, up or down, not maxima. For example,
when "hsv - v" and "sobel x" are applied to the first frame of `challenge_video.mp4` the result is:

  ![alt text][image6]

  The thick highlighted streak to the right is a tar seam, not a lane line. Using pooling correctly highlights the lane lines, but not
  the tar line:

  ![alt text][image7]

* Apply a min threshold to the computed quotients, setting values below the threshold to zero.
* Combine the two channels into a single channel. I add the two channels together and add an extra "bonus point" if both are non-zero.
Here's the Tensorflow code that accomplishes the min thresholding and combining of channels:

  ```
  # Apply min thresholds, and then add the components.
  mask = _tf.greater_equal(vals, _tf.constant((min0, min1), dtype=np.float32))
  mask = _tf.to_float(mask)
  vals = _tf.multiply(vals, mask)
  # We add an extra "bonus point" if both values exceed the min, hence the reduce_prod.
  vals = _tf.reduce_sum(vals, axis=3, keep_dims=True) + _tf.reduce_prod(mask, axis=3, keep_dims=True)
  ```

* Apply a second mean pool to smooth the results.
* Apply a final min threshold.

I then feed the result of the above steps to an instance of the `LaneLineInfo` class to compute the lane lines. The `LaneLineInfo`
processing is a completely separate step and could be refined independently from the computation pipeline described above.

Here is the result of the pipeline on the first frame of `challenge_video.mp4`, which should be compared to the images above:

  ![alt text][image8]

### Finding the Lane Lines from the Pipeline Mask

We'll refer to the result of the computation pipeline as the ***pipeline mask***, even though it is technically not as mask, since its
values are not restricted to being zero or one. Larger values in the pipeline mask are stronger indication of a marking on the
pavement.

The `LaneLineInfo` class in `pipeline.py` handles computing the lane line information from the pipeline mask. This class could
benefit from a lot additional refinement and experimentation. I kept it relatively simple and focused primarily on refining the
computation of the pipeline mask instead.

The only state carried from one frame to the next is:
* The approximate lane width, if known.
* The approximate lane starting x-coordinates.
* The horizontal margin to use for the first search region.

I chose to keep the influence of one frame's computation on the next to a minimum, for a few reasons:
* I wanted to see clearly what the core technique would do on each frame.
* When refining performance on difficult sections of a video, I didn't want to have to run through all
(or a lot of) the prior frames to see the result.
* I found that, for the most part, the computation did well enough, without pursuing such techniques.

The computation is in the `update` method of the `LaneLineInfo` class. It breaks the pipeline mask into
16 horizontal slices, starting at the bottom of the image. For each slice, it computes at most one
'fit point' for each of the left and right lane lines. For each fit point, it also records the 'weight'
of the point. To find a fit point, it computes a left and right search sub-region. It then collects points
in the sub-regions that have a value within 95% of the maximum value in the region. If enough points
are found it computes one 'fit point' from those points. To keep the computation simple and reasonably fast,
it uses the median of the x and y coordinates of the points, rather than trimming outliers and computing
means. When a fit point is found, the x-coordinate of the point is used as the horizontal center of the search
region for the next slice.

If enough fit points are found, a quadratic polynomial is computed (using `np.fitpoly`) from the weighted
fit points.

Of course, there are a lot more details, but they should be clear from the code and comments.

The `LaneLineInfo` class also contains the code to compute the signed curvature at the bottom of the
perspective transformed image, with negative values indicating curvature to the left and positive values
indicating curvature to the right. The curvature values have units `1 / meter`. The absolute value of
such a curvature value is the reciprocal of the radius of curvature.

The `LaneLineInfo` class has properties to return the polynomial coefficients, curvature values, fit
points, and offset of the camera from the center of the computed lane line polynomials. The details
should be clear from the code.


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

---

### Camera Calibration

---

#### Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for camera calibration is contained in `calibrate.py`. This script can be run standalone, via `python calibrate.py`. It reads and processes the
images (provided by Udacity) matched by `./camera_cal/calibration*.jpg`, and saves the pickled values to `./cameraCalibration.p`.

More specifically, the `_gatherPoints` function creates the grid of "object points", then processes each file to gather matching "image points".
It also checks for consistency of the image sizes and logs warnings when either the shapes are inconsistent or `cv2.findChessboardCorners` fails.
The `_gatherPoints` function returns the matching object and image points, as well as the image shape. The `calibrate` function invokes
`cv2.calibrateCamera` and then saves the resulting matrix, distortion values, and image shape in `cameraCalibration.p`.

Here is the console output produced:

```
(carnd-term1) F:\sdc\sdc_proj_4>python calibrate.py
[2017-12-16 17:16:00,538 WARNING calibration]  Couldn't find chess board corners in 'F:\sdc\sdc_proj_4\camera_cal\calibration1.jpg'!
[2017-12-16 17:16:01,829 WARNING calibration]  Inconsistent image shapes, (720, 1280) vs (721, 1281), at 'F:\sdc\sdc_proj_4\camera_cal\calibration15.jpg'! Cropping this image.
[2017-12-16 17:16:03,464 WARNING calibration]  Couldn't find chess board corners in 'F:\sdc\sdc_proj_4\camera_cal\calibration4.jpg'!
[2017-12-16 17:16:03,739 WARNING calibration]  Couldn't find chess board corners in 'F:\sdc\sdc_proj_4\camera_cal\calibration5.jpg'!
[2017-12-16 17:16:04,045 WARNING calibration]  Inconsistent image shapes, (720, 1280) vs (721, 1281), at 'F:\sdc\sdc_proj_4\camera_cal\calibration7.jpg'! Cropping this image.
[2017-12-16 17:16:04,667 INFO calibration]  Gathered corners from 17 images with shape (720, 1280)
[2017-12-16 17:16:05,126 INFO calibration]  Saved camera calibration values to 'F:\sdc\sdc_proj_4\cameraCalibration.p'
[2017-12-16 17:16:05,126 INFO calibration]  Calibration error: 1.1869888548667213
```

Note that finding chess board corners failed for three of the images, and another two images were of size (721, 1281) rather than (720, 1280). The former images
were skipped, while the latter images were cropped to (720, 1280).

Here is the original `calibration2.jpg` image followed by the undistorted version:

![alt text][image9]

![alt text][image10]

The corrected image was generated in `gui_show.py` by loading the `camera_cal` directory (using "Load Pictures..."), scrolling to `calibration2.jpg`, checking the "Undistort" checkbox,
and then clicking "Save Image...".


### Pipeline (single images)

---

#### Provide an example of a distortion-corrected image.

The code to undistort an image is contained in the `getUndistortFunc` function of `pipeline.py`. In particular, the nested `_do` method invokes
`cv2.undistort`. The camera calibration information is loaded from the `cameraCalibration.p` file that was generated by the `calibrate.py` script.
Here's an example original image, together with the result of invoking undistort:

![alt text][image11]
![alt text][image12]

---

#### Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

See the above description of the pipeline. Here is an example ***pipeline mask*** (defined above), computed from `test_images/straight_lines1.jpg`:

![alt text][image13]

Note that this is not a binary image (but has had min thresholds applied), with more intense values indicating stronger signals.

---

#### Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The perspective transformation is implemented by the `Perspective` class in the `pipeline.py` file. Here it is applied to `test_images/straight_lines1.jpg`
with the guide lines visible:

![alt text][image14]

---

#### Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial.

See the above description of the pipeline and finding the lane lines. The polynomial fitting is in the `update` method of the `LaneLineInfo` class
in `pipeline.py`. Here is an example rendering of the lane lines, together with the "fit points", computed from `test_images/test1.jpg`:

![alt text][image15]

---

#### Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The curvature and offset computations are implemented by the `LaneLineInfo` class in `pipeline.py`. Here is the core curvature computation code:

```
  def _curvature(self, coefs):
    """ Returns the signed mathematical curvature. The radius of curvature is the
    absolute value of the reciprocal of this.
    """
    if coefs is None:
      return None

    assert len(coefs) == 3

    # Convert from pixels to meters.
    a = coefs[0] * self._scx / (self._scy * self._scy)
    b = coefs[1] * self._scx / self._scy

    # Note that we use a y-coordinate of zero for the bottom of the image,
    # so we're evaluating derivatives at y = 0.
    return (2 * a) / ((1 + b * b) ** (1.5))
```

The `self._scx` and `self._scy` values are the x and y scaling factors with units meters per pixel. Note that my coordinate system has
zero y value at the ***bottom*** of the (warped) image, not the top, so the derivatives are evaluated at `y = 0`. Note also that this
returns the ***signed curvature***, and ***not*** the ***radius*** of curvature. The radius is the absolute value of the reciprocal of
the return value. In the rendering, I display the signed curvature and the signed radius (reciprocal of the curvature), so the
direction of curvature is clear from the values.

In addition to the left and right curvature, I also display the mean curvature and mean radius, computed as the curvature and radius of
the average of the two polynomials. Note that this is ***not*** the same as averaging the curvature (or radius) values.

Note that the curvature values have units `1 / meter`, while the radius and offset values have units `meter`. The harness application
handles rendering the statistics, in the `_setImage` method of the `Application` class.

---

#### Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The rendering and overlay is implemented in the `_setImage` method of the `Application` class in `gui_show.py`. Here's an example image:

![alt text][image16]

---

### Pipeline (video)

---

#### Provide a link to your final video output. Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./videos/project.mp4). Note that this was saved at 60 frames per second. To easily scroll and view individual frames, load the video in `gui_show.py` (with all transforms turned off).
This video and others are in the `videos` folder.

The 'challenge' video has a few glitches, but does well for most frames. Note that the challenge video was created with the pipeline in "sensitive mode" with the "Use Sensitive Settings" checkbox checked.

The 'harder' video demonstrates poor performance. This was also recorded in "sensitive mode".

---

### Discussion

---

#### Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

See the above descriptions of the pipeline and finding lane lines for a general overview of what I used and why. Here I'll provide a bit more summary information as well as issues encountered.

I started with implementing the camera calibration. This was very straightforward.

I realized that this project would require a significant amount of exploring color spaces, derivative techniques, etc, so decided to invest in a good visualization app, hence,
I implemented `gui_show.py`, with all exploratory transforms in `xforms.py`. Once I'd settled on a general approach, I added `pipeline.py` and hooked it into `gui_show.py`.

In exploring the various test images and videos, I noticed that the V component of HSV generally did a good job of highlighting lane markings, both white and yellow.
I also noticed that derivatives highlighted undesirable features, such as the pavement seams in the challenge video. A little thought suggested finding a
way to highlight local maxima. Neither thresholding nor derivatives would accomplish this directly. Note that using absolute threshold values on V of HSV is problematic
because the thresholds are sensitive to lighting conditions. To locate local maxima, I needed to compute ***relative*** brightness, which, if properly computed, could
have thresholds applied. I realized that a local maximum would be larger than the average of values to either side, so if I divided the V component by the max of the pixel's
left and right mean V values, the ratio should be larger than one, and would be a good measure of brightness relative to its (horizontal) neighbors. This suggesting the
use of mean pooling. Of course, Tensorflow was natural for this, and would perform much better than using numpy. However, Tensorflow doesn't have direct support for
the one-sided pooling that I needed.

I considered two possibilities for one-sided pooling using Tensorflow.
* A ***precise*** method would use convolution with distinct left and right kernels. For example, a left sum of width 5 and height 1 could be computed via (depth-wise) convolution
with padding mode 'SAME' and with a kernel of width 9 containing values `[1 1 1 1 1 0 0 0 0]`, while a corresponding right sum would use values `[0 0 0 0 1 1 1 1 1]`.
To compute the mean, I'd also need a precise count to divide by. Because of the padding, this isn't just '5', but could be computed easily by convolving the same kernel with
an input with all ones. Of course, the denominator values could be computed just once (not on every frame), saving some computation.
* An ***approximate*** method would use mean-pooling ***with*** padding in the y-direction but ***without*** padding in the x-direction, then pad the result on
the left for the left means, and pad on the right for the right means. This has the drawback that the left means are incorrect near the left edge and
the right means are incorrect near the right edge. Some positives are that the kernel width is about half of that needed for the precise option (5 instead of 9), one pooling
is required, rather than two convolutions, and the GPU isn't required to do a multiplication on each pixel. This should be at least 4x faster. Also, this is a bit simpler
to implement.

I chose to use the approximate method for better performance and simplicity. The initial implementation is in the `getComboPooler` function in `xforms.py`.
The final pipeline implementation is in `pipeline.py`.

The `xforms.py` file also contains lots of color space and derivative transform functions, that I wired into the visualization harness. Note that none of the code in
`xforms.py` is used by the pipeline; it contains pure experimentation / visualization code.

Of course, I spent a lot of time experimenting with different color space components, threshold values, etc. Eventually I settled on the V component of HSV
as the primary channel to use, with S of HLS helping with yellow lines on bright (eg, concrete) pavement. The pooling technique avoided being
overly sensitive to lighting conditions, which is a big issue when using absolute threshold values.

With the above, I could compute a good "mask" value from which to find lane lines. I realized that it may help to preserve relative brightness in the "mask",
so my ***pipeline mask*** (defined above), isn't binary.

Finding the lane lines in a robust way is a tricky business, especially on the `harder_challenge_video.mp4`. I didn't spend as much time on this as it needs
to get good performance on that video. I kept this part (implemented in the `LaneLineInfo` class in `pipeline.py`) relatively simple, with very little state
carried from one frame to the next and very little "sanity checking". If I had more time to spend, I would invest heavily in improving this code. For example,
the simplistic approach of the `update` method of `LaneLineInfo` can't possibly perform well with sharp curves, since they involve "horizontal" segments
in the "mask". Sharp curves also beg for a wider field of view. Some form of frame-to-frame smoothing and inference could certainly help when a frame image is
particularly poor because of sudden glare or other short-lived issues. Leveraging a high confidence curve to recognize a portion of the
other lane line would also be beneficial.

It would also make sense to attempt to recognize and distinguish between different kinds of lane lines, eg, solid lines, dashed lines, double lines, lines that
regulate passing zones (solid beside dashed), etc. Of course, the color of lines is also significant and should be carried through, since a yellow line on the
right is a very bad thing, unless you are passing!

Of course, this code also assumes that the vehicle is initially between lane lines, not straddling one. A "real" implementation would need to deal with the possibility
that the vehicle is ***not*** in a proper position relative to the lane lines.
