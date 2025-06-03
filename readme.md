# ROS2 Dataset Bridge

ROS2 dataset bridge to visualize kitti/kitti360/nuscenes datasets in **ROS2** and visualize in **RVIZ2**, come with a GUI for visualization control.

You could checkout the ROS1 version of each visualization package:

- [KITTI ROS1](https://github.com/Owen-Liuyuxuan/kitti_visualize)
- [KITTI360 ROS1](https://github.com/Owen-Liuyuxuan/kitti360_visualize)
- [Nuscenes ROS1](https://github.com/Owen-Liuyuxuan/nuscenes_visualize)

In this repo, we fully re-structure the code and messages formats for ROS2 (humble), and ~~try to~~ integrate the ROS interface for all these three datasets.

## Getting Started:

### Data Preparation

Check the setting up for each dataset. It is ok to have only either one dataset and get the code running. [KITTI](docs/kitti_readme.md), [KITTI360](docs/kitti360_readme.md), [nuscenes](docs/nusc_readme.md).

For customed dataset, it also works as long as it confined with the format of each dataset (mainly tested with KITTI dataset and KITTI360 dataset, because they are easy to construct).

### Software Prerequisite

This repo runs with ROS2 python3 (humble), and we expect PyQt5 correctly setup with ROS installation.

Clone the repo under the {workspace}/src/ folder. Overwrite the folder names in the [launch file](./launch) to point to your data.

```bash
cd ros2_ws/src
git clone https://github.com/Owen-Liuyuxuan/ros2_dataset_bridge
cd ..

# modify and check the data path!! Also control the publishing frequency of the data stream.
nano src/ros2_dataset_bridge/launch/kitti_launch.xml 

colcon build --symlink-install
source install/setup.bash # install/setup.zsh or install/setup.sh for your own need.

# this will launch the data publisher / rviz / GUI controller
ros2 launch ros2_dataset_bridge kitti_launch.xml # kitti360_launch.xml / nuscenes_launch.xml 
```

Notice that as a known issue from [ros2 python package](https://github.com/ros2/launch/issues/187). The launch/rviz files are copied but not symlinked when we run "colcon build",
so **whenever we modify the launch file, we need to rebuild the package**; whenever we want to modify the rviz file, we need to save it explicitly in the src folder.

```bash
colcon build --symlink-install --packages-select=ros2_dataset_bridge # rebuilding only ros2_dataset_bridge
```

### Core Features:

- [x] Full support for Image, LiDAR and bounding boxes. 
- [x] TF-tree (camera and LiDAR).
- [x] GUI control & ROS topic control.


## GUI

![image](docs/gui.png)

### User manual:

    index: integer selection, notice do not overflow the index number.

    Stop: stop any data loading or processing of the visualization node.
    
    Pause: prevent pointer of the sequantial data stream from increasing, keep the current frame.

    Cancel: quit. (click this before killing the entire launch process)

![image](docs/nuscene_visualized.gif)

### ROS Topics

Please check each datasets.

The tf trees are also well constructed. We have a predefined rviz file for visualizing all topics and tf trees.

# 🧾 BEVFormer Data Extraction from NuScenes and CAN Bus

This document describes the fields extracted from the **NuScenes** and **CAN Bus** datasets inside the `_fill_trainval_infos()` function of the `create_data.py` script in the BEVFormer TensorRT pipeline.

---

## 🔹 1. Source: `sample.json`

Each `sample` is a frame containing references to sensor data, annotations, and scene information.

| Extracted Field     | Stored in `info` Key      |
| ------------------- | ------------------------- |
| `token`             | `token`                   |
| `prev`              | `prev`                    |
| `next`              | `next`                    |
| `scene_token`       | `scene_token`             |
| `timestamp`         | `timestamp`               |
| `data['LIDAR_TOP']` | Used to get LiDAR token   |
| `data['CAM_*']`     | Used to get camera tokens |
| `anns`              | Used to fetch annotations |

---

## 🔹 2. Source: `calibrated_sensor.json`

This provides the **extrinsics** between the sensor and the ego frame.

| Extracted Field | Stored in `info` Key              |
| --------------- | --------------------------------- |
| `translation`   | `lidar2ego_translation`           |
| `rotation`      | `lidar2ego_rotation` (quaternion) |

---

## 🔹 3. Source: `ego_pose.json`

Describes the ego vehicle pose in the global frame.

| Extracted Field | Stored in `info` Key               |
| --------------- | ---------------------------------- |
| `translation`   | `ego2global_translation`           |
| `rotation`      | `ego2global_rotation` (quaternion) |

---

## 🔹 4. Source: `sample_data.json`

This file holds metadata about each sensor recording (e.g., LiDAR or camera).

| Extracted Field   | Stored in `info` Key                      |
| ----------------- | ----------------------------------------- |
| `token` (used)    | Used to call `nusc.get_sample_data()`     |
| File path         | `lidar_path`, or `cams[cam]['data_path']` |
| `prev`            | Used to get previous sweeps               |
| Camera intrinsics | `cams[cam]['cam_intrinsic']`              |

---

## 🔹 5. Source: Camera (`CAM_*`) Data

Each sample has six camera images. For each:

* The intrinsic and extrinsic parameters are extracted.
* Used to populate the `cams` dictionary.

| Extracted  | Stored in                     |
| ---------- | ----------------------------- |
| Intrinsics | `cams[cam]["cam_intrinsic"]`  |
| Extrinsics | `cams[cam]` (pose & rotation) |

---

## 🔹 6. Source: `sample_annotation.json` *(only if `test == False`)*

Bounding box annotations and metadata per object in a frame.

| Extracted Field       | Stored in `info` Key                     |
| --------------------- | ---------------------------------------- |
| `token` (used)        | Used to get annotation                   |
| `num_lidar_pts`       | `num_lidar_pts`                          |
| `num_radar_pts`       | `num_radar_pts`                          |
| `category_name`       | `gt_names` (after mapping)               |
| Bounding box geometry | `gt_boxes` (center, size, yaw)           |
| Velocity              | `gt_velocity` (projected to LiDAR frame) |
| Validity flag         | `valid_flag`                             |

---

## 🔹 7. Source: CAN Bus (`can_bus.pkl`)

Accessed via the `_get_can_bus_info()` function. Usually includes ego vehicle motion data.

| Likely Field in CAN Bus | Stored in `info["can_bus"]` |
| ----------------------- | --------------------------- |
| `position`              | `position`                  |
| `orientation`           | `orientation` (quaternion)  |
| `velocity`              | `velocity` or `speed`       |
| `acceleration`          | `acceleration`              |
| `rotation_rate`         | `rotation_rate`             |

---

## 🔹 8. Derived / Computed Fields

These fields are calculated during preprocessing:

| Computed From                         | Stored in `info` Key |
| ------------------------------------- | -------------------- |
| Frame count in current scene          | `frame_idx`          |
| Transforms from previous LiDAR sweeps | `sweeps`             |

---

## ✅ Final `info` Dictionary Keys

| Key                      | Description                                            | Source JSON                   |
| ------------------------ | ------------------------------------------------------ | ----------------------------- |
| `lidar_path`             | File path to top LiDAR frame                           | `sample_data.json`            |
| `token`                  | Unique identifier for the sample                       | `sample.json`                 |
| `prev`, `next`           | Tokens for previous and next frames                    | `sample.json`                 |
| `can_bus`                | Ego vehicle CAN info (velocity, accel, etc.)           | `can_bus.pkl`                 |
| `frame_idx`              | Index of frame in the scene                            | Computed                      |
| `scene_token`            | Scene grouping identifier                              | `sample.json`                 |
| `lidar2ego_translation`  | LiDAR sensor position in ego frame                     | `calibrated_sensor.json`      |
| `lidar2ego_rotation`     | LiDAR sensor rotation in ego frame                     | `calibrated_sensor.json`      |
| `ego2global_translation` | Ego vehicle position in global frame                   | `ego_pose.json`               |
| `ego2global_rotation`    | Ego vehicle rotation in global frame                   | `ego_pose.json`               |
| `timestamp`              | Time when the sample was recorded                      | `sample.json`                 |
| `cams`                   | Dictionary of 6 cameras with intrinsics and extrinsics | `sample_data.json` + computed |
| `sweeps`                 | Past LiDAR sweeps used for temporal modeling           | `sample_data.json`            |
| `gt_boxes`               | GT boxes: \[x, y, z, w, l, h, yaw]                     | `sample_annotation.json`      |
| `gt_names`               | Mapped category names                                  | `sample_annotation.json`      |
| `gt_velocity`            | Object velocities in LiDAR frame                       | `sample_annotation.json`      |
| `num_lidar_pts`          | LIDAR point count per box                              | `sample_annotation.json`      |
| `num_radar_pts`          | RADAR point count per box                              | `sample_annotation.json`      |
| `valid_flag`             | Whether the object has sufficient sensor points        | Computed                      |

---
---
---


### ✅ **Fields from the `.pkl` file (`info`) directly assigned to `input_dict`**

These fields exist in each item of `self.data_infos`, which is loaded from the `.pkl`.

| `result` key             | Source from `.pkl` (`info`)        |
| ------------------------ | ---------------------------------- |
| `sample_idx`             | `info["token"]`                    |
| `pts_filename`           | `info["lidar_path"]`               |
| `sweeps`                 | `info["sweeps"]`                   |
| `ego2global_translation` | `info["ego2global_translation"]`   |
| `ego2global_rotation`    | `info["ego2global_rotation"]`      |
| `prev_idx`               | `info["prev"]`                     |
| `next_idx`               | `info["next"]`                     |
| `scene_token`            | `info["scene_token"]`              |
| `can_bus`                | `info["can_bus"]` (modified later) |
| `frame_idx`              | `info["frame_idx"]`                |
| `timestamp`              | `info["timestamp"] / 1e6`          |

---

### ✅ **Fields conditionally created and added**

These fields are **created and assigned** only if `self.modality['use_camera'] == True`:

| `result` key    | How it's created                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------- |
| `img_filename`  | List of `cam_info['data_path']` from each camera                                               |
| `lidar2img`     | Computed 4×4 matrix: `viewpad @ lidar2cam_rt.T`                                                |
| `cam_intrinsic` | List of 4×4 padded intrinsics: `viewpad[:intrinsic.shape[0], :intrinsic.shape[1]] = intrinsic` |
| `lidar2cam`     | Computed 4×4 transformation matrices from lidar to each camera                                 |

---

### ✅ **Field added only during training mode (`self.test_mode == False`)**

| `result` key | Description                     |
| ------------ | ------------------------------- |
| `ann_info`   | From `self.get_ann_info(index)` |

---

### ✅ **Field modified (but not newly created)**

These are updated based on ego pose:

| Field     | Description                                                                                        |
| --------- | -------------------------------------------------------------------------------------------------- |
| `can_bus` | First 3 values set to `ego2global_translation`, next 4 to quaternion, then 2 yaw angles (rad, deg) |
|           | - Modified: `can_bus[:3] = translation`, `can_bus[3:7] = quaternion`                               |
|           | - Added: `can_bus[-2] = yaw_rad`, `can_bus[-1] = yaw_deg`                                          |

---

### 🔁 `result` becomes `input_dict` and is passed to `self.pipeline(input_dict)`

This triggers `pre_pipeline()` and later the composed transform pipeline (e.g., `MultiScaleFlipAug3D`, etc.), which **creates the remaining fields**.

---

### 🧪 Created Later in `pipeline(input_dict)` or `union2one(queue)`

These are not assigned in `get_data_info()`, but are created later in the data pipeline:

| Field                                 | Description                                           |
| ------------------------------------- | ----------------------------------------------------- |
| `img`                                 | Tensor stack of camera images, created in `union2one` |
| `img_metas`                           | Metadata, including `scene_token`, `can_bus`, etc.    |
| `img_shape`, `ori_shape`, `pad_shape` | Image shape info during transforms                    |
| `scale_factor`, `img_norm_cfg`        | From image normalization and resize                   |
| `img_fields`, `bbox3d_fields`, etc.   | From the format bundle & field collectors             |
| `filename`                            | Image path or point cloud path (depends on collector) |
| `box_type_3d`, `box_mode_3d`          | Set by detection format transforms                    |

These are typically set in transforms like:

* `DefaultFormatBundle3D`
* `Collect3D`
* `NormalizeMultiviewImage`
* `PadMultiViewImage`
* `Resize`, etc.

---