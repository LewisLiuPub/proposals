# A ROS2 Deep Learning Package: ROS/ROS2 acceleration with Heterogeneous AI Computing

## Authors

- Weizhi Liu, wei.zhi.liu@intel.com, wei.zhi.liu@foxmail.com

## Summary

In the recent years, Artificial Intelligence (AI) is widely used in robotic systems and becomes a key factor of robotic, as well it is appealed to develop AI features in an easy way and without adding new hardware.

Thus, it’s a trending for ROS/ROS2 system to adopt mechanisms of bridging deep learning capabilities into diverse robotic functionalities with knowledge separation and enabling the existing hardware by heterogeneous computing.

I introduce here an open-source ROS2 (and ROS) package implementing flexible deep learning inference pipelines. With such features as many types of inputs/outputs, configurable inference tree, it fulfills: “develop once and deploy everywhere”.

## Description

### Design

![Figure-1: ROS AI Pipeline architecture design](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/ROS2_Deep_Learning_package_architecture.PNG)

As shown in Figure 1 above, the open-source ROS2 package implements a deep learning runtime framework and supports dynamic configuration following the design (shown in below picture). We highlight the design rules:

  - Making Deep Learning Features easily adapted to different robotic use scenarios and applications. 
  - AI Pipeline Runtime should support different type of input source. 
  - AI Pipeline can be generated and managed through configuration files.
  - Leverage 3rd-party parsing and optimization tools for supporting different AI framework, performance optimization, and heterogeneous computing.

Under this design, an AI pipeline should be created with 3 parts, connected with fixed sequence:
  1. **Data Input Adapting** should be put as the pipeline’s head, fetching data from different input media, and converting the raw data into specific format. One AI pipeline has and only has one data input adapting source.
  2. **AI Inference matrix** should be put in the middle part of the AI pipeline, following data input adapting part. The inferences can be linked in parallel or cascade into one AI pipeline, as long as the output of the previous inference fits the input of the latter one.
  3. **Inference result output adapting** part gathers the inference results from each inference module connected to it. The inference results can be adapted to ROS topic/service interfaces ro to GUI based visualization (ROS RViz or CV image window).

![Figure-2: AI pipelines lifecycle management](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/impletation_logic.PNG)

Figure-2 above shows the launching process of AI pipelines. AI pipeline can be configured by the pre-designed YAML configuration file, and eventually can be managed by ROS service or other ROS-service based GUI/command-line tools.

This design ensures AI capabilities can be easily used into upper-level robotics applications if the application consumes the output of the package. 

The package can be used in (but not limited to) the below robotic scenarios:
  - Combining with Navigation Stack and Object Analytic add-ons, perform intelligent navigation by detecting moving objects and a weighted inflation layer in Navigation Costmaps. (another proposal of mine will show the details)
  - Adopting AI Object Segmentation and Object Detection capabilities into 2D legacy SLAM or 3D visual SLAM to composite a semantic SLAM.
  - Leveraging the AI features of this package into robotic upper-level applications to fulfill a smart human-robot interactions.

### Current Status

Currently, we choose `Intel OpenVINO toolkit` as the 3rd-party AI Framework parsing and Optimization tool. Intel OpenVINO toolkit takes some advantages:
  - Intel OpenVINO toolkit is a fully open-source, which benefits much for ROS2 community and deep development for AI features and AI use scenarios.
  - Intel OpenVINO toolkit support heterogeneous computing, and optimize the AI inference performance on IA chips.
  - ROS developers are mainly using X86 based work station for robot related development, OpenVINO helps the majority of ROS developers program/test/deploy on the same machine without adding other HW components.

![Figure-3: ROS AI Pipeline implementation based on OpenVINO Toolkit](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/openvino_design_arch.PNG)

#### Types of input source supported:
  - ROS Image topic (sensor_msgs/msg/image)
  - USB camera
  - IP Camera (RTSP Video stream)
  - Realsense Camera (only RGB data is used)
  - Video/Image file (via OpenCV interface)

#### Types of AI inferences supported by now (more are developing):
  - SSD or Yolo based `object detection`
  - Object Segmentation
  - People Analytics (`Face Detection`, `Age-Gender Detection`, `Head-Pose`, `Emotion`, `Face Re-Identification`, `Human Pose`, etc.)
  - Transportation Analytics (`Passenger detection`, `Vehicle detection`, `license plate detection`, `Vehicle Attributes detection`, etc.)

#### Types of AI Inference result output:
Each of AI inferences supports:
  - ros topics
  - CV image window
  - RViz Image topic

### Inference Samples
#### Object Segmentation through Image Topic
- Pipeline Design

![Sample Pipeline for Object Segmentation](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/sample_pipeline_object_segmentation.PNG)

- Pipeline Configuration ([here](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/config_sample_object_segmentation.yaml))

```terminal
Pipelines:
- name: segmentation
  inputs: [ImageTopic]
  infers:
    - name: ObjectSegmentation
      model: /opt/openvino_toolkit/models/segmentation/output/FP16/frozen_inference_graph.xml
      engine: "HETERO:CPU,GPU,MYRIAD"
      label: to/be/set/xxx.labels
      batch: 1
      confidence_threshold: 0.5
  outputs: [ImageWindow, RosTopic, RViz]
  connects:
    - left: ImageTopic
      right: [ObjectSegmentation]
    - left: ObjectSegmentation
      right: [ImageWindow, RosTopic, RViz]
```

- Inference Result Visualizaiton

![Result Visualization for Object Segmentation](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/object_segmentation.gif)

#### Person Analytics through Image File
- Pipeline Design

![Sample Pipeline for Person Analytics](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/sample_pipeline_person_analytics.PNG)

- Pipeline Configuration ([here](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/config_sample_person_analytics.yaml))

```terminal
Pipelines:
- name: people
  inputs: [Image]
  input_path: /opt/openvino_toolkit/ros2_openvino_toolkit/data/images/team.jpg
  infers:
    - name: FaceDetection
      model: /opt/openvino_toolkit/models/face_detection/output/intel/face-detection-adas-0001/FP16/face-detection-adas-0001.xml
      engine: CPU
      label: to/be/set/xxx.labels
      batch: 1
      confidence_threshold: 0.5
      enable_roi_constraint: true # set enable_roi_constraint to false if you don't want to make the inferred ROI (region of interest) constrained into the camera frame
    - name: AgeGenderRecognition
      model: /opt/openvino_toolkit/models/age-gender-recognition/output/intel/age-gender-recognition-retail-0013/FP32/age-gender-recognition-retail-0013.xml
      engine: CPU
      label: to/be/set/xxx.labels
      batch: 16
    - name: EmotionRecognition
      model: /opt/openvino_toolkit/models/emotions-recognition/output/intel/emotions-recognition-retail-0003/FP32/emotions-recognition-retail-0003.xml
      engine: CPU
      label: to/be/set/xxx.labels
      batch: 16
    - name: HeadPoseEstimation
      model: /opt/openvino_toolkit/models/head-pose-estimation/output/intel/head-pose-estimation-adas-0001/FP32/head-pose-estimation-adas-0001.xml
      engine: CPU
      label: to/be/set/xxx.labels
      batch: 16
  outputs: [ImageWindow, RosTopic, RViz]
  connects:
    - left: Image
      right: [FaceDetection]
    - left: FaceDetection
      right: [AgeGenderRecognition, EmotionRecognition, HeadPoseEstimation, ImageWindow, RosTopic, RViz]
    - left: AgeGenderRecognition
      right: [ImageWindow, RosTopic, RViz]
    - left: EmotionRecognition
      right: [ImageWindow, RosTopic, RViz]
    - left: HeadPoseEstimation
      right: [ImageWindow, RosTopic, RViz]
```


- Inference Result Visualizaiton

![Result Visualization for Person Analytics](https://github.com/LewisLiuPub/proposals/blob/main/ROSCon2021/deep_learning_framework/result_person_analytics.PNG)

