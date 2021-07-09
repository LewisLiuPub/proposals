# Intelligent Navigation: Smart Collision Avoidance with Moving Object Detection

## Authors

- Weizhi Liu, wei.zhi.liu@intel.com, wei.zhi.liu@foxmail.com

## Summary

It’s a trend for unmanned systems to share their workspaces with humans in numerous industries and a varied range of applications, along with the requirements of industrial digitalization upgrading.

Under this situation, the intelligence of navigation process is urged for autonomous mobile robots, in order to ensure surroundings (especially humans) safe.

This proposal shows how we adopt deep learning capability to analyze the properties of moving objects, and how we gain the intelligence gene into ROS Navigation stack.

## Description

### Design


This proposal adds AI-accelerated intelligent collision avoidance and scene-based navigation policy.

Below figure shows the pipelines we designed for the proposal.

![Intelligent Navigation Design](https://github.com/LewisLiuPub/proposals/blob/mainintelligent_navigation/intelligent_navigation_design.PNG "Intelligent Navigation Design")

A **RGB-D Camera** is used as the inputs in the design, whose 2D image data (RGB) are streamed to `Deep Learning packages for Object Detection` inference, and whose 3D depth data are streamed to `Object Localization and Tracking` module. The 2D data and 3D data should be aligned by timestamps, making sure both the data capturing the same scenery. 

**Deep Learning Object Detection** module receives 2D image(RGB) data from camera, triggers Object Detection inference process, and outputs Detected Objects and ROI(Range of Interest) information.

**Object Localization and Tracking** module receives 3D depth data from camera, Detected object and ROI from `Deep Learning Object Detection` module and TF (coordination transformation) data from `robot system`, triggers object motion analytics process (tracking the selected objects and localizing them), then outputs the selected moving objects and their motion properties (such as velocity, position, etc.)

**Collision Avoidance Profiling** module receives detected objects info from `Deep Learning Object Detection` module, decides the navigation scenery and collision avoidance policy.

**Costmap for Moving Inflation** is a plugin module for `Navigation Stack`, receiving moving objects data from `Object Localizaiton and Tracking` module and CA Policy from `Collision Avoidance Profiling` module, calculating the new inflation grids for the moving objects and adding the new inflation layer into Costmap layer list.

Below picture shows the details of Moving Inflation.

![Moving Inflation Layer](https://github.com/LewisLiuPub/proposals/blob/mainintelligent_navigation/moving_inflation_layer.PNG "Moving Inflation Layer")

The moving inflation is figured in ellipse shape. The position of the moving object is a focus of the ellipse, the other focus is on the direction of the velocity of the mobbing object. That means, the major axis (the longest diameter) is aligned with the object’s moving direction. The shortest diameter equals the normal inflation diameter, which is adopted in Inflation layer.

The extended moving inflation area depends on two parameters:
- the velocity of the object and,
- the time span of collision impact. 

So, the distance between the ellipse’s foci is simply designed as:
```
Distance between foci = V * ΔT
    V: velocity of the mobbing object
    ΔT: time span of collision impact, configurable parameter.
```

With these add-ons integrated, `navigation stack` can be smarter, since the objects’ motion trends are considered:
  - **Smart path planning**, since the path planned has higher execution ratio, it is not necessary to trigger path re-planning frequently.
  - **Smart collision avoidance**, since the collision avoidance is taken in advance, it brings more time buffer for the robot to avoid collisions. 
  - **Smart motion control**, since the robot have more time to control its movement, the robot can navigate smoothly with mild speed vibration. It can also actively slow down when surroundings are complicated.

### Current Status

We have developed the below pipelines and opensource projects for the idea.

![Current Implementation](https://github.com/LewisLiuPub/proposals/blob/mainintelligent_navigation/current_implementation.PNG "Current Implementation")

#### Key components in our design:
  - Intel Realsense Camera
    - Provides aligned 2D image (RGB) data and 3D depth data, and sends these data into ROS topic interfaces.
    - Source Code: https://github.com/IntelRealSense/realsense-ros for ROS and ROS2

  - ROS/ROS2 OpenVINO Toolkit
      - Provides deep learning object detection inferences and sends inferences results into ROS topic interfaces.
      - Source code
        - https://github.com/intel/ros_openvino_toolkit for ROS
        - https://github.com/intel/ros2_openvino_toolkit for ROS2
  - ROS/ROS2 Object Analytics
    - Provides Object Localization and Object Tracking features, and sends object analytics info into ROS topic interfaces.
    - Source code
      - https://github.com/intel/ros_object_analytics for ROS
      - https://github.com/intel/ros2_object_analytics for ROS2

  - Moving Object
    - Provides Object Motion analytics and Collision Avoidance Policy, and sends object motion info into ROS topic interfaces.
    - Source code
      - https://github.com/intel/ros_moving_object for ROS
      - https://github.com/intel/ros2_moving_object for ROS2

#### Test & Validation

- The protytpe has been created and verified on ROS Navigation Stack.
- Still in progress of porting to ROS2 system.