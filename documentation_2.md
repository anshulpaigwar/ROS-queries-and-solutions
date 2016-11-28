Welcome to the stereovision wiki!

# Pointclouds generation using Point Grey Bumblebee XB3:
![Bumblebee XB3](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/bumblebee.jpg)

The Bumblebee® XB3 is a 3-sensor multi-baseline IEEE-1394b (800Mb/s) stereo camera designed for improved flexibility and accuracy. It features 1.3 mega-pixel sensors and has two baselines available for stereo processing. The extended baseline and high resolution provide more precision at longer ranges, while the narrow baseline improves close range matching and minimum-range limitations.

Bumblebee datasheet: [bumblebee_XB3.pdf](https://www.ptgrey.com/support/downloads/10133/)

##Basic walkthrough:

![PCL from stereo](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/pcl%20from%20stereo.png?raw=true)

##Comparision of various Stereo Matching algorithms:

![compare1](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/comparision1.png?raw=true)

![compare2](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/comparision2.png?raw=true)

## Connecting bumblebee XB3 to your desktop:

As mentioned in the datasheet Bumblebee XB3 has a standard 9-pin IEEE-1394b (also known as [FireWire](https://en.wikipedia.org/wiki/IEEE_1394)) connector that is used for data transmission, camera control and powering the camera. To connect it to the desktop you must have a PCI card, I used _[2-Port OHCI FWB1G-PCI01 1.1 IEEE FireWire 800 PCI Host Adapter Card FWB-PCI02](https://www.amazon.com/2-Port-FWB1G-PCI01-FireWire-Adapter-FWB-PCI02/dp/images/B00DYYPEKU?ie=UTF8&*Version*=1&*entries*=0)_. You must also have a free PCI slot in your motherboard compatible with your card. PCI card I used supports both PCI and PCI-x.

<img src="https://www.firefold.com/Assets/Adapters/FireWire_Adapters/FW-9PF-4PF-ADPT-9P-End.jpg" width="200"><img src="https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/pci%20card.jpg" width="200"><img src="https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/pci-slots.jpg" width="300">

## Installations and setup:
Following setup was used for this experiment:
* Ubuntu 14.04
 
 [ubuntu installation guide](https://help.ubuntu.com/community/Installation)
* ROS Indigo. Make sure you have installed ROS Indigo full desktop since we would require rqt and rviz.
  
 [ROS-Indigo installation guide](http://wiki.ros.org/indigo/Installation/Ubuntu)

## Camera1394stereo:

[Camera1394stereo](http://wiki.ros.org/camera1394stereo) is ROS driver for devices supporting the IEEE 1394 Digital Camera (IIDC) protocol. Download and build the Camera1394stereo package in your catkin_ws.

As mentioned in Camera1394setero wiki you can launch the driver as node using following launch file:
```
<launch>
  <node pkg="camera1394stereo" type="camera1394stereo_node" name="camera1394stereo_node" output="screen" >
    <param name="video_mode" value="format7_mode3" />
    <param name="format7_color_coding" value="raw16" />
    <param name="bayer_pattern" value="grbg" />
    <param name="bayer_method" value="" />
    <param name="stereo_method" value="Interlaced" />
    <param name="camera_info_url_left" value="" />
    <param name="camera_info_url_right" value="" />
  </node>
</launch>
```
once this node is launched, it will publish a interlaced (combined) bayered image and camera info you can check the topic being publised:
```
$ rostopic list
```
You can view the image by:
```
$ rosrun rqt_image_view rqt_image_view image:=/camera/image_raw
```
Or by using rviz

***
As you might have noticed you are not getting three different images rather single interlaced (combined) bayered image. Next step would be to deinterlace this image into three different images but as mentioned in [this](http://answers.ros.org/question/66375/bumblebee-xb3-driver/) thread:

camera1394stereo package, was created to handle binocular cameras, namely bumblebee2 but it should work with any other IIDC camera supported by libdc1394 producing binocular stereo images using the usual way.

From the system point of view, a firewire IIDC stereo camera is seen as a single device (not two or three), it has a unique GID (device identifier on the firewire bus) and a unique set of parameters (frame rate, video mode, ...). Stereo cameras usually use 'smartly' some video modes to transmit 2 or 3 images at a time (one for each camera).

* For 2 images they use a video mode with 16 bit depth (2 images x 8 bits/pixel) or with 32 bit depth (2 images x 16 bits/pixel).

* For 3 images they use a video mode with 24 bit depth (3 images x 8 bits/pixel).
Depending on the camera each single image is either mono or has color information encoded in Bayer pattern indicated by the manufacturer.

camera1394stereo package, deinterleave the stereo image into two separated images and publish them to two different topics (left and right) with the corresponding camera info topic. So in our case it won't work for a stereo camera with 3 images.

However, we can use the following approach. Use a camera1394 nodelet to grab and publish the 24 bit depth images from the bumblebeeXB3 as if it was a mono camera. Set the frame rate and the other camera parameters. And write a nodelet that subscribes to an image topic and deinterlaces the input images, publishing 3 output images to the corresponding output image topics (left, center and right) each one with the corresponding camera info.

Luckily Ravi Chalasani has done this job for us.

### ROS package for Bumblebee XB3:
Download and install the [Bumblebee XB3 package](https://github.com/ravich2-7183/bumblebee_xb3) made by Ravi Chalasani in your catkin_ws.

launch the camera1394 driver node and it will publish a interlaced (combined) bayered image:
```
$ roslaunch bumblebee_xb3 camera1394_24bit.launch
```
Next run the deinterlacer (& debayering) node (this package's node). Note that this node only does deinterlacing. debayering is handled by stereo_image_proc.
```
$ rosrun bumblebee_xb3 bumblebee_xb3_node
```
You will find the warning messages that the calibration files are missing and raw image data is published. Camera calibration is very essential for the stereo vision. You can use the calibration provided with this package. Copy all the calibration files to the directory:
```
$ ~/.ros/camera_info/
```

But I would seriously suggest that you should calibrate your camera yourself and generate your own camera calibration files. Follow the steps below to calibrate cameras in bumblebee XB3:

### Camera calibration:

For the camera calibration we will use ROS image_pipeline package. We will also use it for the generation of point cloud form the stereo image.

#### Step1: Monocular camera calibration:

Follow [this](http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration) tutorial to calibrate individual cameras. 
Start by getting the dependencies and compiling the driver.
```
$ rosdep install camera_calibration
$ rosmake camera_calibration
```
For example to calibrate the center camera you will need to load the corresponding image topic that will be calibrated:
```
$ rosrun camera_calibration cameracalibrator.py --size 8x6 --square 0.108 image:=/camera/center/image_raw camera:=/camera/center
```
You can have different size of checkerboard. After calibration save the calibration files with the same name used by Ravi Chalasani in his package at location: ~/.ros/camerainfo/
You can find names of calibration files made by Ravi in *calibration_files* folder in his package. For center camera its bb_xb3_center.yaml. It should typically look like this:

```
image_width: 1280
image_height: 960
camera_name: bb_xb3_center
camera_matrix:
  rows: 3
  cols: 3
  data: [1630.309095366925, 0, 657.6027000066007, 0, 1626.365758977655, 463.4530493383832, 0, 0, 1]
distortion_model: plumb_bob
distortion_coefficients:
  rows: 1
  cols: 5
  data: [-0.6199174232040585, 0.5667018244379943, 0.001585862717847503, -0.004502668786375525, -0.4910182177499688]
rectification_matrix:
  rows: 3
  cols: 3
  data: [1, 0, 0, 0, 1, 0, 0, 0, 1]
projection_matrix:
  rows: 3
  cols: 4
  data: [1464.81005859375, 0, 656.1179754530458, 0, 0, 1534.274291992188, 461.813496855666, 0, 0, 0, 1, 0]
```  

Ensure that you have also changed the *camera_name* correspondingly inside each calibration file you have generated.

#### Step2: Stereo camera calibration:
Since we have three cameras we can have following combinations for stereo:
* Left and right
* left and center
* center and right

Follow [this](http://wiki.ros.org/camera_calibration/Tutorials/StereoCalibration) for stereo camera calibration. Follow the instructions for naming convention as mentioned in first step.

![calib](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/cam_calib.png?raw=true)

## Pointcloud generation:
Start the stereo_image_proc node in any of the THREE stereo_camera namespaces, say:
```
$ ROS_NAMESPACE=/camera/stereo_camera_LC/ rosrun stereo_image_proc stereo_image_proc
```
You should see that the point cloud topic is now published /camera/stereo_camera_LC/points2
To view the point cloud, start rviz:
```
$ rosrun rviz rviz
```
Add pointcloud2 panel and set the topic /camera/stereo_camera_LC/points2
also change the name of Global frame refrence to stereo_camera_LC.

## Choosing stereo parameter:
Pointclouds visible might not be as you expected and this is because you have not set the stereo parameters yet. Do not panic follow [this](http://wiki.ros.org/stereo_image_proc/Tutorials/ChoosingGoodStereoParameters) tutorial for setting stereo parameters.

Make sure you've built these packages:
```
$ rosmake stereo_image_proc image_view dynamic_reconfigure
```
We will use dynamic_reconfigure to change the stereo processing parameters on the fly. Launch the reconfigure GUI:
```
$ rosrun rqt_reconfigure rqt_reconfigure
```
In the drop-down menu, choose /camera/stereo_camera_LC/stereo_image_proc_XXX. Change the different parameters to get the better pointclouds.

## Results:
![results](https://github.com/anshulpaigwar/ROS-queries-and-solutions/blob/master/wiki/results.png?raw=true)
