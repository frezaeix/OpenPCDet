# My Own Readme 
Explaining different components of OpenPCDet based on its design pattern

# PV-RCNN
![image](https://user-images.githubusercontent.com/13063395/144646058-f0db013f-9391-4ba5-8b88-c2f3397a3910.png)

Based on the image:

0- datasets

a. DataProcessor(object): transform_points_to_voxels using VoxelGeneratorV2 class: important output = batch_dict['voxels'].
* [How does transform_points_to_voxels work?](../master/pcdet/datasets/processor/data_processor.py#L105)
* grid_size = (self.point_cloud_range[3:6] - self.point_cloud_range[0:3]) / np.array(config.VOXEL_SIZE)
* points_to_voxel's outputs:
    * voxels: [M, max_points, ndim] float tensor. only contain points.
    * coordinates: [M, 3] int32 tensor. zyx format.
    * num_points_per_voxel: [M] int32 tensor.


1- VFE step: MeanVFE().forward: Compute **mean of points** in each voxel-> voxel_features
* Visualization example: before vs after

2- 3D SparseConv step: VoxelBackBone8x: takes voxel_features and voxel_coords from step 1. It applies series of sparse conv and returns encoded_spconv_tensor:
* Visualize each step. 
* What does exactly spareconv do? [This link](https://rancheng.github.io/Sparse-Convolution-Explained/) might be helpful.
``` 
   # for detection head
        # [200, 176, 5] -> [200, 176, 2]
        out = self.conv_out(x_conv4)
        
   batch_dict.update({
            'encoded_spconv_tensor': out,
            'encoded_spconv_tensor_stride': 8
        })
        batch_dict.update({
            'multi_scale_3d_features': {
                'x_conv1': x_conv1,
                'x_conv2': x_conv2,
                'x_conv3': x_conv3,
                'x_conv4': x_conv4,
            }
        })
```
* some questions:
   * what are SubMConv3d and SparseConv3d?


3- Map to BEV: HeightCompression:takes batch_dict['encoded_spconv_tensor'] from step 2 and return spatial_features (It stacks 3D feature volume along Z axis).
``` 
encoded_spconv_tensor = batch_dict['encoded_spconv_tensor']
spatial_features = encoded_spconv_tensor.dense()
N, C, D, H, W = spatial_features.shape
spatial_features = spatial_features.view(N, C * D, H, W)
batch_dict['spatial_features'] = spatial_features
batch_dict['spatial_features_stride'] = batch_dict['encoded_spconv_tensor_stride']
``` 
4- VSA step: Voxel Set Abstraction(VSA): VoxelSetAbstraction(nn.Module):
   
   a. Sample points from raw point cloud using FPS -> keypoints
   
   b. Stores interpolated bev features at keypoints. point_bev_features = self.interpolate_from_bev_features using keypoints and batch_dict['spatial_features'] **(why?)** It estimates the feature map value for the given keypoints (x,y) pairs
   
   c. It applies set abstraction module on top of rawpoints using keypoints and stores these new features. There is a pooling module at the end.
   * StackSAModuleMSG(nn.Module): similar to class PointnetSAModuleMSG class in pointnet2 code. there is a pooling here. It takes rawpoints and keypoints then it applies QueryAndGroup, mlps and pooling n times. It returns new_xyz, new_features. (new_xyz: sampled points or keypoints.)**(how does pooling work?)** simply applying max or avg over nsample dimension. 
   * check [this link](https://github.com/frezaeix/Pointnet2_PyTorch#readme) for understanding this module
   
   d. StackSAModuleMSG(nn.Module): this time it takes multi_scale_3d_features and applies QueryAndGroup, mlps and pooling n times then stores these features.
   
   e. Concat the resutls from b, c and d.
   
   f. Applies **self.vsa_point_feature_fusion** (a linear, bn and relu) on the features from prev step.
   
   g. It Returns features from step e and f.
```
      batch_dict['point_features_before_fusion'] = point_features.view(-1, point_features.shape[-1])
      point_features = self.vsa_point_feature_fusion(point_features.view(-1, point_features.shape[-1]))
      batch_dict['point_features'] = point_features  # (BxN, C)
      batch_dict['point_coords'] = point_coords  # (BxN, 4)
```
   
5- Reshape to BEV step: BaseBEVBackbone(nn.Module): 

   a. It takes spatial features from step number 3 then applies some conv+bn+relu and conv transpose+bn+relu. It return these features as data_dict['spatial_features_2d'].
   
6- RPN Head (DenseHead) step: AnchorHeadSingle(AnchorHeadTemplate):

```
AnchorHeadSingle(
  (cls_loss_func): SigmoidFocalClassificationLoss()
  (reg_loss_func): WeightedSmoothL1Loss()
  (dir_loss_func): WeightedCrossEntropyLoss()
  (conv_cls): Conv2d(512, 18, kernel_size=(1, 1), stride=(1, 1))  ## 18 = 6 anchors per location x 3 classes
  (conv_box): Conv2d(512, 42, kernel_size=(1, 1), stride=(1, 1))  ## 42 = 6 anchors x 7: x y z l w h theta
  (conv_dir_cls): Conv2d(512, 12, kernel_size=(1, 1), stride=(1, 1)) ## 6 anchors x 2 NUM_DIR_BINS
)
```
   
   a. It takes spatial_features_2d from step 5 and produces cls, dir, and box predictions
   b. It returns 

```
            data_dict['batch_cls_preds'] = batch_cls_preds
            data_dict['batch_box_preds'] = batch_box_preds
            data_dict['cls_preds_normalized'] = False
```
  
7- Point Head (DenseHead) step: PointHeadSimple(PointHeadTemplate):
   
   a. It takes point_features_before_fusion or point_features from step 4 and produces classification scores.

8- ROI Head step: PVRCNNHead(RoIHeadTemplate):
   
```   
PVRCNNHead(
  (proposal_target_layer): ProposalTargetLayer()
  (reg_loss_func): WeightedSmoothL1Loss()
  (roi_grid_pool_layer): StackSAModuleMSG(
    (groupers): ModuleList(
      (0): QueryAndGroup()
      (1): QueryAndGroup()
    )
    (mlps): ModuleList(....)
  (shared_fc_layer): Sequential(...)
  (cls_layers): Sequential(...)
  (reg_layers): Sequential(...)
)
```

   a. It applies proposal layer:
   
   b. Then it applies roi_grid_pool_layer:
   
   c. Finally it applies cls_layers and reg_layers.

   Inside PVRCNNHead: RoIHeadTemplate-> proposal_layer(self, batch_dict,  nms_config): 
   
```diff
- Its task is to run nms on the proposals. It is class agnostics and multi class version is not implemented.
```

   Inside PVRCNNHead: RoIHeadTemplate-> ProposalTargetLayer(nn.Module): 
   
```diff
- Its task is ...
```

## Backbone 3D
**Related classes:**
1. Point Feature Encoding (PFE). VoxelSetAbstraction(nn.Module):
 *  It contains StackSAModuleMSG(nn.Module) modules.

2. Voxel Feature Encoding (VFE). 
    
    a. VFETemplate(nn.Module)
    
    b. MeanVFE(VFETemplate)
    
    c. PillarVFE(VFETemplate)

3. VoxelBackBone8x(nn.Module)

## Backbone 2D
**Related classes:**
1. Map to BEV: HeightCompression(nn.Module)
2. Map to BEV: PointPillarScatter(nn.Module)
3. BaseBEVBackbone(nn.Module)

## Dense Head
**Related classes:**
1. AnchorHeadTemplate(nn.Module)
* __init__(self, model_cfg, num_class, class_names, grid_size, point_cloud_range, predict_boxes_when_training)
  
  **Example for model_cfg for pvrcnn:**
  {'NAME': 'AnchorHeadSingle', 'CLASS_AGNOSTIC': False, 'USE_DIRECTION_CLASSIFIER': True, 'DIR_OFFSET': 0.78539, 'DIR_LIMIT_OFFSET': 0.0, 'NUM_DIR_BINS': 2,  
   'ANCHOR_GENERATOR_CONFIG': [{'class_name': 'car', 'anchor_sizes': [[4.2, 2.0, 1.6]], 'anchor_rotations': [0, 1.57], 'anchor_bottom_heights': [0],     
                                'align_center': False, 'feature_map_stride': 8, 'matched_threshold': 0.55, 'unmatched_threshold': 0.4}], 
    'TARGET_ASSIGNER_CONFIG': {'NAME': 'AxisAlignedTargetAssigner', 'POS_FRACTION': -1.0, 'SAMPLE_SIZE': 512, 'NORM_BY_NUM_EXAMPLES': False, 
                               'MATCH_HEIGHT':  False, 'BOX_CODER': 'ResidualCoder'}, 
    'LOSS_CONFIG': {'LOSS_WEIGHTS': {'cls_weight': 1.0, 'loc_weight': 2.0, 'dir_weight': 0.2, 'code_weights': [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]}}
  } # End of example
  
  **Example for predict_boxes_when_training for pvrcnn:**
  {'NAME': 'PVRCNNHead', 'CLASS_AGNOSTIC': True, 'SHARED_FC': [256, 256], 'CLS_FC': [256, 256], 'REG_FC': [256, 256], 'DP_RATIO': 0.3, '
   NMS_CONFIG': {'TRAIN': {'NMS_TYPE': 'nms_gpu', 'MULTI_CLASSES_NMS': False, 'NMS_PRE_MAXSIZE': 9000, 'NMS_POST_MAXSIZE': 512, 'NMS_THRESH': 0.8}, 
                  'TEST': {'NMS_TYPE': 'nms_gpu', 'MULTI_CLASSES_NMS': False, 'NMS_PRE_MAXSIZE': 1024, 'NMS_POST_MAXSIZE': 100, 'NMS_THRESH': 0.7}},          
  'ROI_GRID_POOL': {'GRID_SIZE': 6, 'MLPS': [[64, 64], [64, 64]], 'POOL_RADIUS': [0.8, 1.6], 'NSAMPLE': [16, 16], 'POOL_METHOD': 'max_pool'}, 
  'TARGET_CONFIG': {'BOX_CODER': 'ResidualCoder', 'ROI_PER_IMAGE': 128, 'FG_RATIO': 0.5, 'SAMPLE_ROI_BY_EACH_CLASS': True, 'CLS_SCORE_TYPE': 'raw_roi_iou',   
                    'CLS_FG_THRESH': 0.75, 'CLS_BG_THRESH': 0.25, 'CLS_BG_THRESH_LO': 0.1, 'HARD_BG_RATIO': 0.8, 'REG_FG_THRESH': 0.55}, 
    'LOSS_CONFIG': {'CLS_LOSS': 'BinaryCrossEntropy', 'REG_LOSS': 'smooth-l1', 'CORNER_LOSS_REGULARIZATION': True, 
   'LOSS_WEIGHTS': {'rcnn_cls_weight': 1.0,  'rcnn_reg_weight': 1.0, 'rcnn_corner_weight': 1.0, 'code_weights': [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]}}
  } # End of example
  
  **Tasks**
  
  a. Generate anchors
  
  b. Create target_assigner (AxisAlignedTargetAssigner or ATSSTargetAssigner-> Reference: https://arxiv.org/abs/1912.02424)
  
  c. build_losses
  
2. AnchorHeadSingle(AnchorHeadTemplate)

3. PointIntraPartOffsetHead(PointHeadTemplate):  

4. PointHeadSimple(PointHeadTemplate): A simple point-based segmentation head, which are used for PV-RCNN keypoint segmentaion.
    Reference Paper: https://arxiv.org/abs/1912.13192
    PV-RCNN: Point-Voxel Feature Set Abstraction for 3D Object Detection.
    
5. AnchorHeadMulti(AnchorHeadTemplate)

6. PointHeadTemplate(nn.Module):
    
    **Tasks**
  
    a. build_losses(self.model_cfg.LOSS_CONFIG)


## ROI Head
**Related classes:**
1. RoIHeadTemplate(nn.Module)

    **Tasks**
  
    a. Create a target assigner or in fact ProposalTargetLayer(roi_sampler_cfg=self.model_cfg.TARGET_CONFIG)
  
    b. build_losses(self.model_cfg.LOSS_CONFIG)

2. PartA2FCHead(RoIHeadTemplate)

3. PVRCNNHead(RoIHeadTemplate)

    **Tasks**
  
    a. self.roi_grid_pool_layer = pointnet2_stack_modules.StackSAModuleMSG(
            radii=self.model_cfg.ROI_GRID_POOL.POOL_RADIUS,
            nsamples=self.model_cfg.ROI_GRID_POOL.NSAMPLE,
            mlps=mlps,
            use_xyz=True,
            pool_method=self.model_cfg.ROI_GRID_POOL.POOL_METHOD,
       )
        
4. SECONDHead(RoIHeadTemplate)


## Other modules

### PointNet2 modules
**Related Classes: **
1. StackSAModuleMSG(nn.Module)
* __init__(self, *, radii: List[float], nsamples: List[int], mlps: List[List[int]],
                 use_xyz: bool = True, pool_method='max_pool')

**PointNet2 Utils:** 
1. BallQuery(Function)
2. GroupingOperation(Function)
3. QueryAndGroup(nn.Module)
4. FurthestPointSampling(Function)

# Original Readme by the authors of OpenPCDet

<img src="docs/open_mmlab.png" align="right" width="30%">

# OpenPCDet

`OpenPCDet` is a clear, simple, self-contained open source project for LiDAR-based 3D object detection. 

It is also the official code release of [`[PointRCNN]`](https://arxiv.org/abs/1812.04244), [`[Part-A2-Net]`](https://arxiv.org/abs/1907.03670), [`[PV-RCNN]`](https://arxiv.org/abs/1912.13192) and [`[Voxel R-CNN]`](https://arxiv.org/abs/2012.15712). 

**NEW**: `OpenPCDet` has been updated to `v0.5.0` (Dec. 2021).

## Overview
- [Changelog](#changelog)
- [Design Pattern](#openpcdet-design-pattern)
- [Model Zoo](#model-zoo)
- [Installation](docs/INSTALL.md)
- [Quick Demo](docs/DEMO.md)
- [Getting Started](docs/GETTING_STARTED.md)
- [Citation](#citation)


## Changelog
[2021-12-01] **NEW:** `OpenPCDet` v0.5.0 is released with the following features:
* Improve the performance of all models on [Waymo Open Dataset](#waymo-open-dataset-baselines). Note that you need to re-prepare the training/validation data and ground-truth database of Waymo Open Dataset (see [GETTING_STARTED.md](docs/GETTING_STARTED.md)). 
* Support anchor-free [CenterHead](pcdet/models/dense_heads/center_head.py), add configs of `CenterPoint` and `PV-RCNN with CenterHead`.
* Support lastest **PyTorch 1.1~1.10** and **spconv 1.0~2.x**, where **spconv 2.x** should be easy to install with pip and faster than previous version (see the official update of spconv [here](https://github.com/traveller59/spconv)).  
* Support config [`USE_SHARED_MEMORY`](tools/cfgs/dataset_configs/waymo_dataset.yaml) to use shared memory to potentially speed up the training process in case you suffer from an IO problem.  
* Support better and faster [visualization script](tools/visual_utils/open3d_vis_utils.py), and you need to install [Open3D](https://github.com/isl-org/Open3D) firstly. 

[2021-06-08] Added support for the voxel-based 3D object detection model [`Voxel R-CNN`](#KITTI-3D-Object-Detection-Baselines)

[2021-05-14] Added support for the monocular 3D object detection model [`CaDDN`](#KITTI-3D-Object-Detection-Baselines)

[2020-11-27] Bugfixed: Please re-prepare the validation infos of Waymo dataset (version 1.2) if you would like to 
use our provided Waymo evaluation tool (see [PR](https://github.com/open-mmlab/OpenPCDet/pull/383)). 
Note that you do not need to re-prepare the training data and ground-truth database. 

[2020-11-10] The [Waymo Open Dataset](#waymo-open-dataset-baselines) has been supported with state-of-the-art results. Currently we provide the 
configs and results of `SECOND`, `PartA2` and `PV-RCNN` on the Waymo Open Dataset, and more models could be easily supported by modifying their dataset configs. 

[2020-08-10] Bugfixed: The provided NuScenes models have been updated to fix the loading bugs. Please redownload it if you need to use the pretrained NuScenes models.

[2020-07-30] `OpenPCDet` v0.3.0 is released with the following features:
   * The Point-based and Anchor-Free models ([`PointRCNN`](#KITTI-3D-Object-Detection-Baselines), [`PartA2-Free`](#KITTI-3D-Object-Detection-Baselines)) are supported now.
   * The NuScenes dataset is supported with strong baseline results ([`SECOND-MultiHead (CBGS)`](#NuScenes-3D-Object-Detection-Baselines) and [`PointPillar-MultiHead`](#NuScenes-3D-Object-Detection-Baselines)).
   * High efficiency than last version, support **PyTorch 1.1~1.7** and **spconv 1.0~1.2** simultaneously.
   
[2020-07-17]  Add simple visualization codes and a quick demo to test with custom data. 

[2020-06-24] `OpenPCDet` v0.2.0 is released with pretty new structures to support more models and datasets. 

[2020-03-16] `OpenPCDet` v0.1.0 is released. 


## Introduction


### What does `OpenPCDet` toolbox do?

Note that we have upgrated `PCDet` from `v0.1` to `v0.2` with pretty new structures to support various datasets and models.

`OpenPCDet` is a general PyTorch-based codebase for 3D object detection from point cloud. 
It currently supports multiple state-of-the-art 3D object detection methods with highly refactored codes for both one-stage and two-stage 3D detection frameworks.

Based on `OpenPCDet` toolbox, we win the Waymo Open Dataset challenge in [3D Detection](https://waymo.com/open/challenges/3d-detection/), 
[3D Tracking](https://waymo.com/open/challenges/3d-tracking/), [Domain Adaptation](https://waymo.com/open/challenges/domain-adaptation/) 
three tracks among all LiDAR-only methods, and the Waymo related models will be released to `OpenPCDet` soon.    

We are actively updating this repo currently, and more datasets and models will be supported soon. 
Contributions are also welcomed. 

### `OpenPCDet` design pattern

* Data-Model separation with unified point cloud coordinate for easily extending to custom datasets:
<p align="center">
  <img src="docs/dataset_vs_model.png" width="95%" height="320">
</p>

* Unified 3D box definition: (x, y, z, dx, dy, dz, heading).

* Flexible and clear model structure to easily support various 3D detection models: 
<p align="center">
  <img src="docs/model_framework.png" width="95%">
</p>

* Support various models within one framework as: 
<p align="center">
  <img src="docs/multiple_models_demo.png" width="95%">
</p>


### Currently Supported Features

- [x] Support both one-stage and two-stage 3D object detection frameworks
- [x] Support distributed training & testing with multiple GPUs and multiple machines
- [x] Support multiple heads on different scales to detect different classes
- [x] Support stacked version set abstraction to encode various number of points in different scenes
- [x] Support Adaptive Training Sample Selection (ATSS) for target assignment
- [x] Support RoI-aware point cloud pooling & RoI-grid point cloud pooling
- [x] Support GPU version 3D IoU calculation and rotated NMS 


## Model Zoo

### KITTI 3D Object Detection Baselines
Selected supported methods are shown in the below table. The results are the 3D detection performance of moderate difficulty on the *val* set of KITTI dataset.
* All models are trained with 8 GTX 1080Ti GPUs and are available for download. 
* The training time is measured with 8 TITAN XP GPUs and PyTorch 1.5.

|                                             | training time | Car@R11 | Pedestrian@R11 | Cyclist@R11  | download | 
|---------------------------------------------|----------:|:-------:|:-------:|:-------:|:---------:|
| [PointPillar](tools/cfgs/kitti_models/pointpillar.yaml) |~1.2 hours| 77.28 | 52.29 | 62.68 | [model-18M](https://drive.google.com/file/d/1wMxWTpU1qUoY3DsCH31WJmvJxcjFXKlm/view?usp=sharing) | 
| [SECOND](tools/cfgs/kitti_models/second.yaml)       |  ~1.7 hours  | 78.62 | 52.98 | 67.15 | [model-20M](https://drive.google.com/file/d/1-01zsPOsqanZQqIIyy7FpNXStL3y4jdR/view?usp=sharing) |
| [SECOND-IoU](tools/cfgs/kitti_models/second_iou.yaml)       | -  | 79.09 | 55.74 | 71.31 | [model-46M](https://drive.google.com/file/d/1AQkeNs4bxhvhDQ-5sEo_yvQUlfo73lsW/view?usp=sharing) |
| [PointRCNN](tools/cfgs/kitti_models/pointrcnn.yaml) | ~3 hours | 78.70 | 54.41 | 72.11 | [model-16M](https://drive.google.com/file/d/1BCX9wMn-GYAfSOPpyxf6Iv6fc0qKLSiU/view?usp=sharing)| 
| [PointRCNN-IoU](tools/cfgs/kitti_models/pointrcnn_iou.yaml) | ~3 hours | 78.75 | 58.32 | 71.34 | [model-16M](https://drive.google.com/file/d/1V0vNZ3lAHpEEt0MlT80eL2f41K2tHm_D/view?usp=sharing)|
| [Part-A2-Free](tools/cfgs/kitti_models/PartA2_free.yaml)   | ~3.8 hours| 78.72 | 65.99 | 74.29 | [model-226M](https://drive.google.com/file/d/1lcUUxF8mJgZ_e-tZhP1XNQtTBuC-R0zr/view?usp=sharing) |
| [Part-A2-Anchor](tools/cfgs/kitti_models/PartA2.yaml)    | ~4.3 hours| 79.40 | 60.05 | 69.90 | [model-244M](https://drive.google.com/file/d/10GK1aCkLqxGNeX3lVu8cLZyE0G8002hY/view?usp=sharing) |
| [PV-RCNN](tools/cfgs/kitti_models/pv_rcnn.yaml) | ~5 hours| 83.61 | 57.90 | 70.47 | [model-50M](https://drive.google.com/file/d/1lIOq4Hxr0W3qsX83ilQv0nk1Cls6KAr-/view?usp=sharing) |
| [Voxel R-CNN (Car)](tools/cfgs/kitti_models/voxel_rcnn_car.yaml) | ~2.2 hours| 84.54 | - | - | [model-28M](https://drive.google.com/file/d/19_jiAeGLz7V0wNjSJw4cKmMjdm5EW5By/view?usp=sharing) |
||
| [CaDDN (Mono)](tools/cfgs/kitti_models/CaDDN.yaml) |~15 hours| 21.38 | 13.02 | 9.76 | [model-774M](https://drive.google.com/file/d/1OQTO2PtXT8GGr35W9m2GZGuqgb6fyU1V/view?usp=sharing) |

### Waymo Open Dataset Baselines
We provide the setting of [`DATA_CONFIG.SAMPLED_INTERVAL`](tools/cfgs/dataset_configs/waymo_dataset.yaml) on the Waymo Open Dataset (WOD) to subsample partial samples for training and evaluation, 
so you could also play with WOD by setting a smaller `DATA_CONFIG.SAMPLED_INTERVAL` even if you only have limited GPU resources. 

By default, all models are trained with **20% data (~32k frames)** of all the training samples on 8 GTX 1080Ti GPUs, and the results of each cell here are mAP/mAPH calculated by the official Waymo evaluation metrics on the **whole** validation set (version 1.2).    

|    Performance@(train with 20\% Data)            | Vec_L1 | Vec_L2 | Ped_L1 | Ped_L2 | Cyc_L1 | Cyc_L2 |  
|---------------------------------------------|----------:|:-------:|:-------:|:-------:|:-------:|:-------:|
| [SECOND](tools/cfgs/waymo_models/second.yaml) | 70.96/70.34|62.58/62.02|65.23/54.24	|57.22/47.49|	57.13/55.62 |	54.97/53.53 | 
| [CenterPoint](tools/cfgs/waymo_models/centerpoint_without_resnet.yaml)| 71.33/70.76|63.16/62.65|	72.09/65.49	|64.27/58.23|	68.68/67.39	|66.11/64.87|
| [CenterPoint (ResNet)](tools/cfgs/waymo_models/centerpoint.yaml)|72.76/72.23|64.91/64.42	|74.19/67.96	|66.03/60.34|	71.04/69.79	|68.49/67.28 |
| [Part-A2-Anchor](tools/cfgs/waymo_models/PartA2.yaml) | 74.66/74.12	|65.82/65.32	|71.71/62.24	|62.46/54.06	|66.53/65.18	|64.05/62.75 |
| [PV-RCNN (AnchorHead)](tools/cfgs/waymo_models/pv_rcnn.yaml) | 75.41/74.74	|67.44/66.80	|71.98/61.24	|63.70/53.95	|65.88/64.25	|63.39/61.82 | 
| [PV-RCNN (CenterHead)](tools/cfgs/waymo_models/pv_rcnn_with_centerhead_rpn.yaml) | 75.95/75.43	|68.02/67.54	|75.94/69.40	|67.66/61.62	|70.18/68.98	|67.73/66.57|

We could not provide the above pretrained models due to [Waymo Dataset License Agreement](https://waymo.com/open/terms/), 
but you could easily achieve similar performance by training with the default configs.

### NuScenes 3D Object Detection Baselines
All models are trained with 8 GTX 1080Ti GPUs and are available for download.

|                                             | mATE | mASE | mAOE | mAVE | mAAE | mAP | NDS | download | 
|---------------------------------------------|----------:|:-------:|:-------:|:-------:|:---------:|:-------:|:-------:|:---------:|
| [PointPillar-MultiHead](tools/cfgs/nuscenes_models/cbgs_pp_multihead.yaml) | 33.87	| 26.00 | 32.07	| 28.74 | 20.15 | 44.63 | 58.23	 | [model-23M](https://drive.google.com/file/d/1p-501mTWsq0G9RzroTWSXreIMyTUUpBM/view?usp=sharing) | 
| [SECOND-MultiHead (CBGS)](tools/cfgs/nuscenes_models/cbgs_second_multihead.yaml) | 31.15 |	25.51 |	26.64 | 26.26 | 20.46 | 50.59 | 62.29 | [model-35M](https://drive.google.com/file/d/1bNzcOnE3u9iooBFMk2xK7HqhdeQ_nwTq/view?usp=sharing) |



### Other datasets
Welcome to support other datasets by submitting pull request. 

## Installation

Please refer to [INSTALL.md](docs/INSTALL.md) for the installation of `OpenPCDet`.


## Quick Demo
Please refer to [DEMO.md](docs/DEMO.md) for a quick demo to test with a pretrained model and 
visualize the predicted results on your custom data or the original KITTI data.

## Getting Started

Please refer to [GETTING_STARTED.md](docs/GETTING_STARTED.md) to learn more usage about this project.


## License

`OpenPCDet` is released under the [Apache 2.0 license](LICENSE).

## Acknowledgement
`OpenPCDet` is an open source project for LiDAR-based 3D scene perception that supports multiple
LiDAR-based perception models as shown above. Some parts of `PCDet` are learned from the official released codes of the above supported methods. 
We would like to thank for their proposed methods and the official implementation.   

We hope that this repo could serve as a strong and flexible codebase to benefit the research community by speeding up the process of reimplementing previous works and/or developing new methods.


## Citation 
If you find this project useful in your research, please consider cite:


```
@misc{openpcdet2020,
    title={OpenPCDet: An Open-source Toolbox for 3D Object Detection from Point Clouds},
    author={OpenPCDet Development Team},
    howpublished = {\url{https://github.com/open-mmlab/OpenPCDet}},
    year={2020}
}
```

## Contribution
Welcome to be a member of the OpenPCDet development team by contributing to this repo, and feel free to contact us for any potential contributions. 


