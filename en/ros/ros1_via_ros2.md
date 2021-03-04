# ROS (1) via ROS 2 Bridge (User Guide)

UNDER CONSTRUCTION

This topic explains how to setup and use ROS 2 with PX4.
It provides an overview of the ROS2-PX4 bridge architecture and application pipeline, along with instructions on how to install all the needed software and build ROS 2 applications.

**In this guide you will learn:**
1. How to connect ROS 2 nodes with the PX4 Autopilot, using the [microRTPS bridge](../middleware/micrortps.md), and the [`px4_ros_com`](https://github.com/PX4/px4_ros_com) package.
2. How to connect ROS (ROS "version 1") nodes with the PX4 Autopilot by using the [`ros1_bridge`](https://github.com/ros2/ros1_bridge) package (by Open Robotics) to bridge ROS 2 and ROS (1).

:::note
The Fast DDS interface in the PX4 Autopilot can be leveraged by any applications running and linked in DDS domains (including ROS nodes).

For information about using the *microRTPS bridge* without ROS 2, see the [RTPS/DDS Interface section](../middleware/micrortps.md).
:::

:::note
For a more detailed and visual explanation on how to use PX4 with ROS 2 see these presentations from the PX4 maintainers:
1. [ROS World 2020 - Getting started with ROS 2 and PX4](https://www.youtube.com/watch?v=qhLATrkA_Gw)
1. [PX4 Dev Summit 2019 - "ROS 2 Powered PX4"](https://www.youtube.com/watch?v=2Szw8Pk3Z0Q)
:::

## ROS (1) Application Pipeline

The architecture for integrating ROS (1) applications with the PX4 Autopilot is shown below.

:::note
It's important to note the use of the [ros1_bridge](https://github.com/ros2/ros1_bridge) (by Open Robotics), which bridges messages between ROS 2 and ROS.

This is needed because the first version of ROS does not support RTPS.
:::

![Architecture with ROS](../../assets/middleware/micrortps/architecture_ros.png)

## Code Generation

### Dependencies and Requirements

- Follow [Fast DDS Installation Guide](../dev_setup/fast-dds-installation.md) to install **Fast RTPS(DDS)** and **Fast-RTPS-Gen**.
- 



### ROS 2 / ROS (1) Applications

The [`px4_ros_com`](https://github.com/PX4/px4_ros_com) package, when built, generates everything needed to access the PX4 Autopilot uORB messages from a ROS 2 node (for ROS (1) you also need [ros1_bridge](https://github.com/ros2/ros1_bridge)).
This includes all the required components of the *microRTPS bridge*, including the `micrortps_agent` and the IDL files (required by `micrortps_agent`).

The ROS (1) and ROS 2 message definition headers and interfaces are generated from the [px4_msgs](https://github.com/PX4/px4_msgs) package, which match the uORB messages counterparts under PX4 Autopilot.
These are required by `px4_ros_com` when generating the IDL files to be used by the `micrortps_agent`.

Both `px4_ros_com` and `px4_msgs` packages have two separate branches, `master` & `ros1`:
- The `master` branch, used with ROS 2, contains code to generate all the required ROS 2 messages and IDL files to bridge the PX4 Autopilot with ROS 2 nodes.
- The `ros1` branch, used with ROS, contains code to generate the ROS message headers and source files, which can be used **with** the `ros1_bridge` to share data between the PX4 Autopilot and ROS (1).
- both branches include some example listener and advertiser example nodes

## Agent interfacing with a ROS 2 middleware

Building `px4_ros_com` automatically generates and builds the agent application, though it requires (as a dependency), that the `px4_msgs` package also gets built on the same ROS 2 workspace (or overlaid from another ROS 2 workspace), and since it is also installed using the [`colcon`](http://design.ros2.org/articles/build_tool.html) build tools, running it works exactly the same way as the above.

See the [Building the `px4_ros_com` package](#building-the-px4-ros-com-and-px4-msgs-packages) for details about the build structure.

## Supported uORB messages

The generated bridge code will enable a specified subset of uORB topics to be published/subscribed via RTPS. This is true for both ROS or non-ROS applications.

For *automatic code generation* there's a *yaml* definition file in the PX4 **PX4-Autopilot/msg/tools/** directory called **uorb_rtps_message_ids.yaml**. This file defines the set of uORB messages to be used with RTPS, whether the messages are to be sent, received or both, and the RTPS ID for the message to be used in ROS 2 middleware.

Thanks to an API change in **"ROS 2 Dashing"** we now use the `rosidl_generate_interfaces()` CMake module (in `px4_msgs`) to generate the IDL files that we require for microRTPS agent generation (in `px4_ros_com`).

The PX4 Autopilot includes a template for the IDL file generation, which is only used during the PX4 build process.

The `px4_msgs` build generates *slightly different* IDL files for use with ROS 2/ROS (than are built for PX4 Autopilot firmware).

The **uorb_rtps_message_ids.yaml** is transformed in a way that the message names become *PascalCased* (the name change is irrelevant to the client-agent communication, but is critical for ROS 2, since the message naming must follow the PascalCase convention).

The new IDL files also reverse the messages that are sent and received (required because if a message is sent from the client side, then it's received on the agent side, and vice-versa).

:::note
It's important that an RTPS identifier (id) is set for all messages.
:::

```yaml
rtps:
  - msg: ActuatorArmed
    id: 0
  - msg: ActuatorControl
    id: 1
  - ...
  - msg: Airspeed
    id: 5
    send: true
  - msg: BatteryStatus
    id: 6
    send: true
  - msg: CameraCapture
    id: 7
  - msg: CameraTrigger
    id: 8
    receive: true
  - ...
  - msg: SensorBaro
    id: 63
    receive: true
    send: true
```

## Building the `px4_ros_com` and `px4_msgs` packages

Install and setup the ROS 2 environment on your development machine and separately clone the `px4_ros_com` and `px4_msgs` repo from the `master` branch.


:::note
For ROS (1) environments, you need separately clone, checkout, install and setup both the `master`and `ros1` branches (see [above for more information](#px4_ros_com)).
:::

### Installing ROS 2 and Dependencies

:::note
This install and build guide covers ROS 2 Foxy in Ubuntu 20.04.
:::

1. [Install ROS 2 Foxy](https://index.ros.org/doc/ros2/Installation/Foxy/Linux-Install-Debians/)
1. The install process should also install the **`colcon`** build tools, but in case that doesn't happen, you can install the tools manually:

   ```sh
   sudo apt install python3-colcon-common-extensions
   ```

1. **`eigen3_cmake_module`** is also required, since Eigen3 is used on the transforms library:

   ```sh
   sudo apt install ros-dashing-eigen3-cmake-module
   ```

1. Some Python dependencies must also be installed (using **`pip`** or **`apt`**):

   ```sh
   sudo pip3 install -U empy pyros-genmsg setuptools
   ```

   :::warning
   If you plan to use ROS 1 topics, in communication with ROS 2, you need to build the `ros1_bridge` from source,
   and setup a separate workspace.

   Check the section: **Setting up the workspaces > ROS 1 workspace**.
   :::

### Setting up the workspaces

Since the ROS 2 and ROS require different environments you will need a separate workspace for each ROS version.The directory `px4_ros_com/scripts` contains multiple scripts that can be used to build both workspaces.

:::tip
The commands shown below are only examples, and while work when followed, we are making the assumption that you want to install and clone in your home directory. It's up to you decide where you want your source to live on your development host.
:::

#### ROS 2 workspace

Create a workspace using:
```sh
$ mkdir -p ~/px4_ros_com_ros2/src
```

Clone the respective ROS 2 (`master`) branch to the `/src` directory:
```sh
$ git clone https://github.com/PX4/px4_ros_com.git ~/px4_ros_com_ros2/src/px4_ros_com # clones the master branch
$ git clone https://github.com/PX4/px4_msgs.git ~/px4_ros_com_ros2/src/px4_msgs
```
#### ROS (1) workspace

For ROS, follow exactly the same process, but create a different directory and clone a different branch:
```sh
mkdir -p ~/px4_ros_com_ros1/src
```

Then, clone the respective ROS (1) (`ros1`) branch to the `/src` directory:

```sh
$ git clone https://github.com/PX4/px4_ros_com.git ~/px4_ros_com_ros1/src/px4_ros_com -b ros1 # clones the 'ros1' branch
$ git clone https://github.com/PX4/px4_msgs.git ~/px4_ros_com_ros1/src/px4_msgs -b ros1
```
#### Building the Workspace

:::tip
Make sure you check all the available options for the available build scripts with the `--help` argument.

- Using the `--verbose` argument will allow you to see the full *colcon* build output.
- Each build script has it's own helpful usage argments
:::

:::note
The build process described below (for all workspaces) will open new tabs on the console, corresponding to different stages of the build process that need to have different environment configurations sourced.
:::

#### Building ROS 2 Workspace
Use the `build_ros2_workspace.bash` script to build the ROS 2 workspace to where the `px4_ros_com` and `px4_msgs` `master` branches were cloned.

```sh
$ cd scripts
$ source build_ros2_workspace.bash
```

#### Building ROS (1) Workspace
Make sure you are on the `ros1` branch of `px4_ros_com`, and use the `build_ros1_bridge.bash`, to build the `ros1_bridge`, to build the ROS (1) workspace to where the `px4_ros_com` and `px4_msgs` `ros1` branches were cloned.

```sh
$ git checkout ros1
$ cd scripts
$ source build_ros1_bridge.bash
```

#### Building both ROS (1) & ROS 2 Workspaces

To build both (ROS (1) and ROS 2) workspaces with a single script, use the `build_all.bash`. The most common way of using it, is by passing the ROS(1) workspace directory path and also the PX4 Autopilot directory path:

```sh
$ source build_all.bash --ros1_ws_dir <path/to/px4_ros_com_ros1/ws>
```

:::note
Remember you need a separate clone of the `px4_ros_com` repository with the `ros1` branch checked-out.
:::

#### Custom builds (for reference)
The steps below show how to *manually* build the packages (provided for your information/better understanding only).

**To build the ROS 2 workspace only:**

1. `cd` into `px4_ros_com_ros2` dir and source the ROS 2 environment.
   Don't mind if it tells you that a previous workspace was set before:

   ```sh
   cd ~/px4_ros_com_ros2
   source /opt/ros/foxy/setup.bash
   ```

2. Build the workspace:

   ```sh
   colcon build --symlink-install --event-handlers console_direct+
   ```        

To build both workspaces (in replacement of the previous steps):

1. `cd` into `px4_ros_com_ros2` dir and source the ROS 2 environment.
   Don't mind if it tells you that a previous workspace was set before:

   ```sh
   source /opt/ros/foxy/setup.bash
   ```

1. Clone the `ros1_bridge` package so it can be built on the ROS 2 workspace:

   ```sh
   git clone https://github.com/ros2/ros1_bridge.git -b dashing ~/px4_ros_com_ros2/src/ros1_bridge
   ```

1. Build the `px4_ros_com` and `px4_msgs` packages, excluding the `ros1_bridge` package:

   ```sh
   colcon build --symlink-install --packages-skip ros1_bridge --event-handlers console_direct+
   ```

   :::note
   `--event-handlers console_direct+` only serves the purpose of adding verbosity to the `colcon` build process and can be removed if one wants a more "quiet" build.
   :::

1. Then, follows the process of building the ROS(1) packages side.
   For that, one requires to open a new terminal window and source the ROS(1) environment that has installed on the system:

   ```sh
   source /opt/ros/melodic/setup.bash
   ```

1. On the terminal of the previous step, build the `px4_ros_com` and `px4_msgs` packages on the ROS end:

   ```sh
   cd ~/px4_ros_com_ros1 && colcon build --symlink-install --event-handlers console_direct+
   ```

1. Before building the `ros1_bridge`, one needs to open a new terminal and then source the environments and workspaces following the order below:

   ```sh
   source ~/px4_ros_com_ros1/install/setup.bash
   source ~/px4_ros_com_ros2/install/setup.bash
   ```

1. Finally, build the `ros1_bridge`.
   Note that the build process may consume a lot of memory resources.
   On a resource limited machine, reduce the number of jobs being processed in parallel (e.g. set environment variable `MAKEFLAGS=-j1`).
   For more details on the build process, see the build instructions on the [ros1_bridge](https://github.com/ros2/ros1_bridge) package page.
   ```sh
   cd ~/px4_ros_com_ros2 && colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure --event-handlers console_direct+
   ```

### Cleaning the workspaces

After building the workspaces there are many files that must be deleted before you can do a clean/fresh build (for example, after you have changed some code and want to rebuild).

Unfortunately *colcon* does not currently have a way of cleaning the generated **build**, **install** and **log** directories, so these directories must be deleted manually.

The **clean_all.bash** script (in **px4_ros_com/scripts**) is provided to ease this cleaning process, this script can be used to clean all of the workspace options listed above (ROS 2, ROS 1, and Both)

The most common way of using it is by passing it the ROS(1) workspace directory path (since it's usually not on the default path):

```sh
$ source clean_all.bash --ros1_ws_dir <path/to/px4_ros_com_ros1/ws>
```

:::tip
Similar to the build scripts, the `clean_all.bash` script also has a `--help` guide.
:::

<!-- 
## Creating a ROS 2 listener

With the `px4_ros_com` built successfully, one can now take advantage of the generated *microRTPS* agent app and also from the generated sources and headers of the ROS 2 msgs from `px4_msgs`, which represent a one-to-one matching with the uORB counterparts. 

To create a listener node on ROS 2, lets take as an example the `sensor_combined_listener.cpp` node under `px4_ros_com/src/examples/listeners`:

....
-->

<!--

## Creating a ROS 2 advertiser

A ROS 2 advertiser node publishes data into the DDS/RTPS network (and hence to the PX4 Autopilot).
...
-->

## Creating a ROS (1) listener

Since the creation of ROS nodes is a well known and documented process, we are going to leave this section out from this guide, and you can find an example of a ROS listener for `sensor_combined` messages the `ros1` branch of the `px4_ros_com` repository, under the following path `src/listeners/`.

## Offboard control

For a complete reference example on how to use Offboard control with PX4, see: [ROS 2 Offboard control example](../ros/ros2_offboard_control.md).

## Quick-start testing the bridge with PX4 SITL

To quickly test the package (using PX4 SITL with Gazebo), make sure you have checked-out the latest PX4 source from GitHub, and [setup your development environment](../dev_setup/dev_env.md).

1. From the root of the PX4 Autopilot project, start SITL Gazebo using:
   ```sh
   make px4_sitl_rtps gazebo
   ```

### To only use ROS 2:

1. Make sure to source the workspace configuration first:

   ```sh
   source ~/px4_ros_com_ros2/install/setup.bash
   ```

1. On a terminal, source the ROS 2 workspace and then start the `micrortps_agent` daemon with UDP as the transport protocol:
   ```sh
   $ source ~/px4_ros_com_ros2/install/setup.bash
   $ micrortps_agent -t UDP
   ```

1. On the [NuttShell/System Console](../debug/system_console.md) of the SITL window, start the `micrortps_client` daemon also in UDP:
   ```sh
   pxh> micrortps_client start -t UDP
   ```

### To use ROS (1) and ROS 2:

1. On one terminal, source the ROS 2 environment and workspace and launch the `ros1_bridge` (this allows ROS 2 and ROS nodes to communicate with each other).
   Also set the `ROS_MASTER_URI` where the `roscore` is/will be running:
   ```sh
   $ source /opt/ros/dashing/setup.bash
   $ source ~/px4_ros_com_ros2/install/local_setup.bash
   $ export ROS_MASTER_URI=http://localhost:11311
   $ ros2 run ros1_bridge dynamic_bridge
   ```

1. On another terminal, source the ROS workspace and launch the `sensor_combined` listener node.
   Since you are launching through `roslaunch`, this will also automatically start the `roscore`:
   ```sh
   $ source ~/px4_ros_com_ros1/install/setup.bash
   $ roslaunch px4_ros_com sensor_combined_listener.launch
   ```

1. On a terminal, source the ROS 2 workspace and then start the `micrortps_agent` daemon with UDP as the transport protocol:
   ```sh
   $ source ~/px4_ros_com_ros2/install/setup.bash
   $ micrortps_agent -t UDP
   ```

1. On the [NuttShell/System Console](../debug/system_console.md), start the `micrortps_client` daemon also in UDP:
   ```sh
   > micrortps_client start -t UDP
   ```

For both cases above, in case of success. you will be able to see the data being printed on the terminal/console where you launched the ROS listener:

   ```sh
   RECEIVED DATA FROM SENSOR COMBINED
   ================================
   ts: 870938190
   gyro_rad[0]: 0.00341645
   gyro_rad[1]: 0.00626475
   gyro_rad[2]: -0.000515705
   gyro_integral_dt: 4739
   accelerometer_timestamp_relative: 0
   accelerometer_m_s2[0]: -0.273381
   accelerometer_m_s2[1]: 0.0949186
   accelerometer_m_s2[2]: -9.76044
   accelerometer_integral_dt: 4739
   ```

You can also verify the rate of the message using `rostopic hz`.
   For the case of `sensor_combined`:
   ```sh
   average rate: 248.187
   	min: 0.000s max: 0.012s std dev: 0.00147s window: 2724
   average rate: 248.006
   	min: 0.000s max: 0.012s std dev: 0.00147s window: 2972
   average rate: 247.330
   	min: 0.000s max: 0.012s std dev: 0.00148s window: 3212
   average rate: 247.497
   	min: 0.000s max: 0.012s std dev: 0.00149s window: 3464
   average rate: 247.458
   	min: 0.000s max: 0.012s std dev: 0.00149s window: 3712
   average rate: 247.485
   	min: 0.000s max: 0.012s std dev: 0.00148s window: 3960
   ```

You can also test the `sensor_combined` ROS 2 listener by launching the example launch file in the terminal, and it should also get data being printed to the console output.

   ```sh
   $ source ~/px4_ros_com_ros2/install/setup.bash
   $ ros2 launch px4_ros_com sensor_combined_listener.launch.py
   ```

:::note
When using the `build_all.bash` script, it automatically opens and sources all the required terminals, so you just have to run the respective apps in each terminal.
:::

## Additional information

* [DDS and ROS middleware implementations](https://github.com/ros2/ros2/wiki/DDS-and-ROS-middleware-implementations)