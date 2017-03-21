
#Advanced Lane Finding Project

The goal of this project is to write a software pipeline to identify lane boundaries. As input we take a video stream from a forward-facing camera mounted on the front of a car, and we will output a annotated video.
The code for this is contained in the IPython notebook located in "Advanced Lane Finding.ipynb". 

[//]: # (Image References)

[image1]: ./examples/rep_undist.png "Undistorted"
[image2]: ./examples/rep_warped.png "Warp Example"
[image3]: ./examples/rep_binary.png "Binary" 
[image4]: ./examples/hist.png "Histogram"
[image5]: ./examples/rep_out.png "Output"
[video1]: ./result.mp4 "Video"


### Step 1: Camera Calibration and Distortion Correction

##### The first step in the project is to remove any distortion from the images by calculating the camera calibration matrix and distortion coefficients using a series of images of a chessboard.


I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

Focus on the edges of the images to see the effect of the distortion correction.


### Step 2: Perspective Transform
##### In this step a perspective transform to convert the  binary image into "birds-eye view" is applied.

The `perspective_transform()` function takes as inputs an image (`img`) and converts it tos bird view. We can invert this transformation by setting the inverse parameter to True. 
To make the perspective transform work I had to hardcode source (`src_coords`) and destination (`dst_coords`) points.  I chose to  hard code the source and destination points in the following manner:

```

    src_coords = np.float32(
        [[580, 460],
         [200, 720],
         [706, 460],
         [1140, 720]])
    
    dst_coords = np.float32(
        [[200, 100],
         [200, 720],
         [1040, 100],
         [1040, 720]])

```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image2]


### Step 3: Apply Color and Gradient Thresholds
##### I used a combination of color and gradient thresholds to generate a binary image. 

First I made use of the Sobel operator
to detect edges. The Sobel operator is at the heart of the Canny edge detection algorithm. Applying the Sobel operator to an image is a way of taking the derivative of the image in the x or y direction. For our lane detection algorithm I took the gradient in the x direction, because it  emphasizes edges closer to vertical. Lane lines tend to be vertical. 
This was implemented with OpenCV's `cv2.Sobel` function, which works with a single color channel. So I had to convert the images to grayscale beforehand.

After that I applied a color threshold. Therefore I converted the image data to HLS and also HSV color space. I separate the S channel (from HLS) and the V channel (from HSV). In HLS/HSV color space we can represent color independent of any change in brightness. In our test images a combination of S and V channel  did best at picking up lanes. 

As output I created a binary images that combines the color and sobel threshold.
Here is in example output:

![alt text][image3]


### Step 4: Identify lane Pixels and Fit Polynomial
##### In this step we will try to extract the actual lane pixels.

After all our preprocessing we should have a binary image where the lane lines stand out clearly. However, we still need to decide explicitly which pixels are part of the lines and which belong to the left line and which belong to the right line.

I first take a histogram along all the columns in the lower half of the image. The result looks like this:

![alt text][image4]

With this histogram I am adding up the pixel values along each column in the image. In my thresholded binary image, pixels are either 0 or 1, so the two most prominent peaks in this histogram will be good indicators of the x-position of the base of the lane lines. I can use that as a starting point for where to search for the lines. From that point, I can use a sliding window, placed around the line centers, to find and follow the lines up to the top of the frame.

Once I have identified pixels for left and right lanes I can use the Numpy polyfit() function to fit a polynomial equation to each lane. 

```
    left_fit = np.polyfit(lefty, leftx, 2)
    right_fit = np.polyfit(righty, rightx, 2)
```


### Step 5: Radius of Curvature and Vehicle Position
##### Here I  calculate the radius of curvature for each lane and the vehicle position with respect to the center of the lane

I created a helper function ` radius_of_curvature(xvals, yvals)` to calculate the radius of curvature. Inside the function first I fit a circle to the my lane pixel values using `np.polyfit`
after that I use this formula to calculate the radius 

```
curverad = ((1 + (2*fit_cr[0]*np.max(yvals) + fit_cr[1])**2)**1.5) / np.absolute(2*fit_cr[0])

```

Take a closer look at this [awesome tutorial](http://www.intmath.com/applications-differentiation/8-radius-curvature.php) to get some intuition for the formula.
We calculate the radius of curvature for both lanes and take the average as result.


After that I checked weather my result makes sense. Therfore I compared my result with the U.S. government specifications for highway curvature. Effectively, this checks to see that the calculated lane is not turning faster than we would expect.

Next I calculate where the lane intercects the bottom of the image. Check the `get_interceptss(polynomial)` helper function for this. We already have the fitted a polynomial to our lane lines so all we have to to is fill in max/min y values to get the bottom/top intercepts.
With the intercepts we can calulate the postion of the vehicle. We assume the camera is mounted on the center of the car and does not move. So we can simply add the intercept x values and devide by 2. After converting from pixel space to meter space we have our final result.

When working with the video stream I average the intercept values to get smoother lane detection.

I fill the space between the left and right lane line with `cv2.fillPoly` to get a better visualisation.
Here is an example result:

![alt text][image5]


---

### Final Result Video Pipeline 

Here's a [link][video1] to my video result

---

###Discussion

My pipeline performs pretty well on the test data but does fail on the challenge data. There is room for improvement in the color thresholding step. My pipeline is not robust enough to shadows.


