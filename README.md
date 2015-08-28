# Bridge communication between ROS 1 and ROS 2

This package provides a network bridge which enables to exchange messages between ROS 1 and ROS 2.

The bridge is currently implemented in C++ simply because there is no Python API for ROS 2 yet.
Therefore it is limited to all the message types available at compile time.


## Prerequisites

In order to run the bridge you need to either:

* get [prebuilt binaries](https://github.com/ros2/ros2/releases) or
* build the bridge as well as the other ROS 2 packages from source.

After that you can run both examples described below.

For all examples you need to source the environment of the ament install space where the bridge was built in or unpacked to.
Additionally you will need to either source the ROS 1 environment or at least set the `ROS_MASTER_URI` and run a `roscore`.

The following ROS 1 packages are required to build and use the bridge:
* `catkin`
* `roscpp`
* `rosmsg`
* `std_msgs`
* as well as the Python package `rospkg`


### Build the bridge from source

Before continuing you should have ROS 2 built from source following [these](https://github.com/ros2/ros2/wiki) instructions.

**Note:** you need almost 4 GB of free RAM per thread to compile the bridge.
This will be fixed once `rclcpp` has been [refactored into a library](https://github.com/ros2/rclcpp/issues/48).

The bridge uses `pkg-config` to find ROS 1 packages.
ROS 2 packages are located in CMake using `find_package()`.
Therefore the `CMAKE_PREFIX_PATH` must not contain paths from ROS 1 which would overlay ROS 2 packages.

In order to build this ament package you must first setup the ROS 1 environment:

```
. /opt/ros/indigo/setup.bash
unset CMAKE_PREFIX_PATH
```

and second source the ROS 2 environment:

```
. <workspace-with-ros2>/install/setup.bash
```

Afterwards you can build the ament package as usual.
Due to the high memory usage you might want to restrict the number of parallel jobs.

**Note:** due to a bug in ROS 1 rosbag you currently must patch an installed file or build ROS 1 `ros_comm` from source.
Edit the file `/opt/ros/indigo/lib/python2.7/dist-packages/rosbag/bag.py` with root permissions and comment out the line containing `import roslz4`.
Make sure to revert the change after building the bridge.
The next ROS 1 patch release will resolve this problem (https://github.com/ros/ros_comm/pull/642).

```
ament build <workspace-with-bridge> --make-args -j1
```


## Example 1: run the bridge and the example talker and listener

The talker and listener can be either a ROS 1 or a ROS 2 node.
The bridge will pass the message along transparently.


### Example 1a: ROS 1 talker and ROS 2 listener

First we start a ROS 1 `roscore`:

```
# Shell A:
. /opt/ros/indigo/setup.bash
roscore
```

---

Then we start the dynamic bridge which will watch the available ROS 1 and ROS 2 topics.
Once a *matching* topic has been detected it starts to bridge the messages on this topic.

```
# Shell B:
. <install-space-with-bridge>/setup.bash
export ROS_MASTER_URI=http://localhost:11311
dynamic_bridge
```

The program will start outputting the currently available topics in ROS 1 and ROS 2 in a regular interval.

---

Now we start the ROS 1 talker.

```
# Shell C:
. /opt/ros/indigo/setup.bash
rosrun rospy_tutorials talker
```

The ROS 1 node will start printing the published messages to the console.

---

Now we start the ROS 2 listener.

```
# Shell D:
. <install-space-with-ros2>/setup.bash
listener
```

The ROS 2 node will start printing the received messages to the console.

When looking at the output in *shell B* there will be a line stating that the bridge for this topic has been created:

```
created 1to2 bridge for topic 'chatter' with ROS 1 type 'std_msgs/String' and ROS 2 type 'std_msgs/String'
```

At the end stop all programs with `Ctrl-C`.
Once you stop either the talker or the listener in *shell B* a line will be stating that the bridge has been teared down:

```
removed 1to2 bridge for topic 'chatter'
```

The screenshot shows all the shell windows and their expected content:

![ROS 1 talker and ROS 2 listener](doc/ros1_talker_ros2_listener.png)


### Example 1b: ROS 2 talker and ROS 1 listener

The steps are very similar to the previous example and therefore only the commands are described.

```
# Shell A:
. /opt/ros/indigo/setup.bash
roscore
```

---

```
# Shell B:
. <install-space-with-bridge>/setup.bash
export ROS_MASTER_URI=http://localhost:11311
dynamic_bridge
```

---

Now we start the ROS 2 talker.

```
# Shell C:
. <install-space-with-ros2>/setup.bash
talker
```

---

Now we start the ROS 1 listener.

```
# Shell D:
. /opt/ros/indigo/setup.bash
rosrun roscpp_tutorials listener
```


## Example 2: run the bridge and exchange images

The second example will demonstrate the bridge passing along bigger and more complicated messages.
A ROS 2 node is publishing images retrieved from a camera and on the ROS 1 side we use `rqt_image_view` to render the images in a GUI.
And a ROS 1 publisher can send a message to toggle an option in the ROS 2 node.

First we start a ROS 1 `roscore` and the bridge:

```
# Shell A:
. /opt/ros/indigo/setup.bash
roscore
```

```
# Shell B:
. <workspace-with-bridge>/install/setup.bash
export ROS_MASTER_URI=http://localhost:11311
dynamic_bridge
```

---

Now we start the ROS 1 GUI:

```
# Shell C:
. /opt/ros/indigo/setup.bash
rqt_image_view
```

**Note:** The currently released version of `rqt_image_view` does not allow to specify the topic name on the command line.
And since the GUI only shows available ROS 1 topics and for resource reasons the bridge only starts bridging messages once publishers and subscribers are available on both sides the ROS 2 topic won't be listed.
Therefore we have to briefly start a ROS 1 publisher to workaround the limitation of `rqt_image_view`.

```
# Shell D:
. /opt/ros/indigo/setup.bash
rosrun usb_cam usb_cam_node usb_cam/image_raw:=image
```

While the `usb_cam_node` process is running you should be able to select the `/image` topic in `rqt_image_view` and see the current images.
Now we can stop the `usb_cam_node` process in *shell D* with Ctrl+C

--

Now we start the ROS 2 image publisher:
```
# Shell E:
. <workspace-with-ros2>/install/setup.bash
cam2image
```

You should see the current images in `rqt_image_view` which are coming from the ROS 2 node `cam2image` and are being passed along by the bridge.

--

To exercide the bridge in the opposite direction at the same time you can publish.
You can either use the `Message Publisher` plugin in `rqt` to publish a `std_msgs/Bool` message on the topic `flip_image`.
By publishing either `true` or `false` the camera node with conditionally flip the image before sending it.

As an alternative you can also run one of the two following `rostopic` commands:

```
# Shell F:
. /opt/ros/indigo/setup.bash
rostopic pub -r 1 /flip_image std_msgs/Bool "{data: true}"
rostopic pub -r 1 /flip_image std_msgs/Bool "{data: false}"
```

The screenshot shows all the shell windows and their expected content:

![ROS 2 camera and ROS 1 rqt](doc/ros2_camera_ros1_rqt.png)