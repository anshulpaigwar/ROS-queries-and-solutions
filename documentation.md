###First thing first:
Go to BIOS setting and turn off **Secure Boot** option. Secure Boot prevents access to Ubuntu kernel files and hence causes Nvidia driver installation to fail.

###Verify You Have a CUDA-Capable GPU:
````
lspci | grep -i nvidia
````
If you do not see any settings, update the PCI hardware database that Linux maintains by entering update-pciids (generally found in /sbin) at the command line and rerun the previous lspci command.

If your graphics card is from NVIDIA and it is listed in [http://developer.nvidia.com/cuda-gpus](http://developer.nvidia.com/cuda-gpus), your GPU is CUDA-capable.

### Easy way(Installation using Debian file):

The easiest method is to visit the [CUDA Downloads](https://developer.nvidia.com/cuda-downloads) web page and download the deb (network) file that matches your Ubuntu. Installing from Debian file is as easy as:
````
$ sudo apt-get install build-essential
$ cd Downloads
$ sudo dpkg -i cuda-repo-ubuntu1404_7.5-18_amd64.deb
$ sudo apt-get update
$ sudo apt-get install cuda
````
Reboot the system.

**Environment Variables:**

As part of the CUDA environment, you should add the following in the .bashrc file of your home folder.

````
export CUDA_HOME=/usr/local/cuda-7.5 
export LD_LIBRARY_PATH=${CUDA_HOME}/lib64 
 
PATH=${CUDA_HOME}/bin:${PATH} 
export PATH
````
To add this to .bashrc use: `$ gedit ~/.bashrc`

**Test your Cuda installation:**

Now you can copy the SDK samples into your home directory, and build a test sample.
````
$ mkdir ~/cuda_samples
$ cp -a ~/usr/local/cuda-7.5/samples/. ~/cuda_samples
$ cd ~/cuda_samples/1_Utilities/deviceQuery
$ make
$ ./deviceQuery
````
If you are lucky enough and everything is fine device characteristics should be displayed.
 ***

### Installation using .run file:

Other not so lucky people like me would have to face following problems:

1. Some people might face a **Login Loop** problem: You start Ubuntu and you get the graphical login, but it is being displayed at an extremely low resolution (like 640x480 for example). You login, you see the desktop for a second, something fails and you are thrown back to the graphical login screen.

2. For the people having **Nvidia Geforce Graphics cards with Optimus technology** above stuff may not work and would get error that no Cuda capable device are found on running `./deviceQuery`.

The problem is that installation through `.deb` debian file is not supported so you must install using `.run` file. A detailed description about the process can be found at following links:

[link-1](https://codeyarns.com/2015/09/25/how-to-install-cuda-7-5-on-ubuntu/), [link-2](http://askubuntu.com/questions/451672/installing-and-testing-cuda-in-ubuntu-14-04), [link-3](https://devtalk.nvidia.com/default/topic/755994/-quot-no-cuda-capable-device-is-detected-quot-with-cuda-gpu-attached/).

You must remove all the Nvidia drivers and Cuda and Xorg files completely:
````
$ sudo apt-get remove --auto-remove nvidia-cuda-toolkit
$ sudo apt-get purge --auto-remove nvidia-cuda-toolkit
$ sudo apt-get remove --purge nvidia-*
$ sudo rm /etc/X11/xorg.conf
````
Visit the [CUDA Downloads](https://developer.nvidia.com/cuda-downloads) web page and download the runfile (local) installation file.

*Now take the photo of following commands or open this web page in another device so that you could refer them.*

In order to install from .run file we must kill X process i.e we have to shut GUI. Hit CTL+ALT+F1 to drop to the physical terminal and log in.
````
$ cd Downloads
$ sudo service lightdm stop
$ sudo killall Xorg
$ chmod +x cuda_7.5.18_linux.run
$ sudo sh cuda_7.5.18_linux.run
````
Now just follow all the command instructions. and choose to install Nvidia drivers and cuda both. For people using both integrated graphics card and Nvidia graphics card **Make sure you choose not to install OpenGL libraries.**

test if the drivers are working by going to your sample directory `cd /usr/local/cuda/samples` and copy them to your local directories. follow all the process described above to test Cuda installation. Run `./deviceQuery` device characteristics should now be displayed. **cheer up!**

Start the GUI

````
$ sudo service lightdm start
````

Setup Environment variables as described above.

***
## Install OpenCV with cuda:

**OpenCV version: 2.4.9**

Follow [this](http://blog.aicry.com/ubuntu-14-04-install-opencv-with-cuda/) tutorial to install opencv with cuda.

You might get following error when you execute `make` command:
````
Unsupported GPU architecture 'compute_11' CMake Error at
````
Then you have to manually specify your compute capability, which you can find [here](https://developer.nvidia.com/cuda-gpus). To resolve issue just add `-D CUDA_ARCH_BIN=3.0` to your CMAKE arguments: (My GPU compute capability is 3.0)

* Note: I am not installing QT because it creates confilcts with emotion team (INRIA) packages. 
````
$ cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local -D WITH_TBB=ON -D BUILD_NEW_PYTHON_SUPPORT=ON -D WITH_V4L=ON -D INSTALL_C_EXAMPLES=ON -D INSTALL_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_QT=OFF -D WITH_OPENGL=ON -D ENABLE_FAST_MATH=1 -D CUDA_FAST_MATH=1 -D WITH_CUBLAS=1 -D CUDA_ARCH_BIN=3.0 .. 
````

### Patching OpenCV:

The OpenCV version 2.4.9 and cuda version 6.5 and higher are not compatible. It requires a few modifications in OpenCV. Edit the file <opencv_dir>/modules/gpu/src/nvidia/core/NCVPixelOperations.hpp in the opencv source code main directory. Remove the keyword "static" from lines 51 to 58, 61 to 68 and from 119 to 133 in it. This is an example: 
Before:
````cpp 	
template<> static inline __host__ __device__ Ncv8u  _pixMaxVal<Ncv8u>()  {return UCHAR_MAX;}
template<> static inline __host__ __device__ Ncv16u _pixMaxVal<Ncv16u>() {return USHRT_MAX;}
````
After:
````cpp
template<> inline __host__ __device__ Ncv8u  _pixMaxVal<Ncv8u>()  {return UCHAR_MAX;}
template<> inline __host__ __device__ Ncv16u _pixMaxVal<Ncv16u>() {return USHRT_MAX;}
````
Now you can execute `make` command and error should be resovled.

## ROS indigo installation:

For more info check ROS indigo installation [tutorial](http://wiki.ros.org/indigo/Installation/Ubuntu).
Setup your computer to accept software from packages.ros.org. ROS Indigo ONLY supports Saucy (13.10) and Trusty (14.04) for Debian packages.

````
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
````

Set up your keys using following command (the command available on ros tutorial doesn't seem to work):
````
$ wget https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -O - | sudo apt-key add -
````

Then:
````
$ sudo apt-get update
$ sudo apt-get install ros-indigo-desktop-full
$ sudo rosdep init
$ rosdep update
$ echo "source /opt/ros/indigo/setup.bash" >> ~/.bashrc
$ source ~/.bashrc
$ sudo apt-get install python-rosinstall
````
