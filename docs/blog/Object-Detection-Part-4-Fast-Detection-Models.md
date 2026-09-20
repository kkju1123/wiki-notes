---
title: 'Object Detection Part 4: Fast Detection Models'
url: https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/
source_type: web
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:47:05.544188+00:00'
---

In [Part 3](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/), we have reviewed models in the R-CNN family. All of them are region-based object detection algorithms. They can achieve high accuracy but could be too slow for certain applications such as autonomous driving. In Part 4, we only focus on fast object detection models, including SSD, RetinaNet, and models in the YOLO family.

Links to all the posts in the series: [[Part 1](https://lilianweng.github.io/posts/2017-10-29-object-recognition-part-1/)] [[Part 2](https://lilianweng.github.io/posts/2017-12-15-object-recognition-part-2/)] [[Part 3](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/)] [[Part 4](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/)].

## Two-stage vs One-stage Detectors

Models in the R-CNN family are all region-based. The detection happens in two stages: (1) First, the model proposes a set of regions of interests by select search or regional proposal network. The proposed regions are sparse as the potential bounding box candidates can be infinite. (2) Then a classifier only processes the region candidates.

The other different approach skips the region proposal stage and runs detection directly over a dense sampling of possible locations. This is how a one-stage object detection algorithm works. This is faster and simpler, but might potentially drag down the performance a bit.

All the models introduced in this post are one-stage detectors.

## YOLO: You Only Look Once

The **YOLO** model (**“You Only Look Once”**; [Redmon et al., 2016](https://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Redmon_You_Only_Look_CVPR_2016_paper.pdf)) is the very first attempt at building a fast real-time object detector. Because YOLO does not undergo the region proposal step and only predicts over a limited number of bounding boxes, it is able to do inference super fast.

## Workflow

1.   **Pre-train** a CNN network on image classification task.

2.   Split an image into cells. If an object’s center falls into a cell, that cell is “responsible” for detecting the existence of that object. Each cell predicts (a) the location of bounding boxes, (b) a confidence score, and (c) a probability of object class conditioned on the existence of an object in the bounding box.

    *   The **coordinates** of bounding box are defined by a tuple of 4 values, (center x-coord, center y-coord, width, height) — , where and are set to be offset of a cell location. Moreover, , , and are normalized by the image width and height, and thus all between (0, 1].
    *   A **confidence score** indicates the likelihood that the cell contains an object: `Pr(containing an object) x IoU(pred, truth)`; where `Pr` = probability and `IoU` = interaction under union.
    *   If the cell contains an object, it predicts a **probability** of this object belonging to every class : `Pr(the object belongs to the class C_i | containing an object)`. At this stage, the model only predicts one set of class probabilities per cell, regardless of the number of bounding boxes, .
    *   In total, one image contains bounding boxes, each box corresponding to 4 location predictions, 1 confidence score, and K conditional probabilities for object classification. The total prediction values for one image is , which is the tensor shape of the final conv layer of the model.

3.   The final layer of the pre-trained CNN is modified to output a prediction tensor of size .

![Image 1](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/yolo.png)

The workflow of YOLO model. (Image source: [original paper](https://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Redmon_You_Only_Look_CVPR_2016_paper.pdf))

## Network Architecture

The base model is similar to [GoogLeNet](https://www.cs.unc.edu/~wliu/papers/GoogLeNet.pdf) with inception module replaced by 1x1 and 3x3 conv layers. The final prediction of shape is produced by two fully connected layers over the whole conv feature map.

![Image 2](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/yolo-network-architecture.png)

The network architecture of YOLO.

## Loss Function

The loss consists of two parts, the _localization loss_ for bounding box offset prediction and the _classification loss_ for conditional class probabilities. Both parts are computed as the sum of squared errors. Two scale parameters are used to control how much we want to increase the loss from bounding box coordinate predictions () and how much we want to decrease the loss of confidence score predictions for boxes without objects (). Down-weighting the loss contributed by background boxes is important as most of the bounding boxes involve no instance. In the paper, the model sets and .

> NOTE: In the original YOLO paper, the loss function uses instead of as confidence score. I made the correction based on my own understanding, since every bounding box should have its own confidence score. Please kindly let me if you do not agree. Many thanks.

where,

*   : An indicator function of whether the cell i contains an object.
*   : It indicates whether the j-th bounding box of the cell i is “responsible” for the object prediction (see Fig. 3).
*   : The confidence score of cell i, `Pr(containing an object) * IoU(pred, truth)`.
*   : The predicted confidence score.
*   : The set of all classes.
*   : The conditional probability of whether cell i contains an object of class .
*   : The predicted conditional class probability.

![Image 3](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/yolo-responsible-predictor.png)

At one location, in cell i, the model proposes B bounding box candidates and the one that has highest overlap with the ground truth is the "responsible" predictor.

The loss function only penalizes classification error if an object is present in that grid cell, . It also only penalizes bounding box coordinate error if that predictor is “responsible” for the ground truth box, .

As a one-stage object detector, YOLO is super fast, but it is not good at recognizing irregularly shaped objects or a group of small objects due to a limited number of bounding box candidates.

## SSD: Single Shot MultiBox Detector

The **Single Shot Detector** (**SSD**; [Liu et al, 2016](https://arxiv.org/abs/1512.02325)) is one of the first attempts at using convolutional neural network’s pyramidal feature hierarchy for efficient detection of objects of various sizes.

## Image Pyramid

SSD uses the [VGG-16](https://arxiv.org/abs/1409.1556) model pre-trained on ImageNet as its base model for extracting useful image features. On top of VGG16, SSD adds several conv feature layers of decreasing sizes. They can be seen as a _pyramid representation_ of images at different scales. Intuitively large fine-grained feature maps at earlier levels are good at capturing small objects and small coarse-grained feature maps can detect large objects well. In SSD, the detection happens in every pyramidal layer, targeting at objects of various sizes.

![Image 4](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/SSD-architecture.png)

The model architecture of SSD.

## Workflow

Unlike YOLO, SSD does not split the image into grids of arbitrary size but predicts offset of predefined _anchor boxes_ (this is called “default boxes” in the paper) for every location of the feature map. Each box has a fixed size and position relative to its corresponding cell. All the anchor boxes tile the whole feature map in a convolutional manner.

Feature maps at different levels have different receptive field sizes. The anchor boxes on different levels are rescaled so that one feature map is only responsible for objects at one particular scale. For example, in Fig. 5 the dog can only be detected in the 4x4 feature map (higher level) while the cat is just captured by the 8x8 feature map (lower level).

![Image 5](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/SSD-framework.png)

The SSD framework. (a) The training data contains images and ground truth boxes for every object. (b) In a fine-grained feature maps (8 x 8), the anchor boxes of different aspect ratios correspond to smaller area of the raw input. (c) In a coarse-grained feature map (4 x 4), the anchor boxes cover larger area of the raw input. (Image source: [original paper](https://arxiv.org/abs/1512.02325))

The width, height and the center location of an anchor box are all normalized to be (0, 1). At a location of the -th feature layer of size , , we have a unique linear scale proportional to the layer level and 5 different box aspect ratios (width-to-height ratios), in addition to a special scale (why we need this? the paper didn’t explain. maybe just a heuristic trick) when the aspect ratio is 1. This gives us 6 anchor boxes in total per feature cell.

![Image 6](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/SSD-box-scales.png)

An example of how the anchor box size is scaled up with the layer index for . Only the boxes of aspect ratio are illustrated.

At every location, the model outputs 4 offsets and class probabilities by applying a conv filter (where is the number of channels in the feature map) for every one of anchor boxes. Therefore, given a feature map of size , we need prediction filters.

## Loss Function

Same as YOLO, the loss function is the sum of a localization loss and a classification loss.

where is the number of matched bounding boxes and balances the weights between two losses, picked by cross validation.

The _localization loss_ is a [smooth L1 loss](https://github.com/rbgirshick/py-faster-rcnn/files/764206/SmoothL1Loss.1.pdf) between the predicted bounding box correction and the true values. The coordinate correction transformation is same as what [R-CNN](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/#r-cnn) does in [bounding box regression](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/#bounding-box-regression).

where indicates whether the -th bounding box with coordinates is matched to the -th ground truth box with coordinates for any object. are the predicted correction terms. See [this](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/#bounding-box-regression) for how the transformation works.

The _classification loss_ is a softmax loss over multiple classes ([softmax_cross_entropy_with_logits](https://www.tensorflow.org/api_docs/python/tf/nn/softmax_cross_entropy_with_logits) in tensorflow):

where indicates whether the -th bounding box and the -th ground truth box are matched for an object in class . is the set of matched bounding boxes ( items in total) and is the set of negative examples. SSD uses [hard negative mining](https://lilianweng.github.io/posts/2017-12-31-object-recognition-part-3/#common-tricks) to select easily misclassified negative examples to construct this set: Once all the anchor boxes are sorted by objectiveness confidence score, the model picks the top candidates for training so that neg:pos is at most 3:1.

## YOLOv2 / YOLO9000

**YOLOv2** ([Redmon & Farhadi, 2017](https://arxiv.org/abs/1612.08242)) is an enhanced version of YOLO. **YOLO9000** is built on top of YOLOv2 but trained with joint dataset combining the COCO detection dataset and the top 9000 classes from ImageNet.

## YOLOv2 Improvement

A variety of modifications are applied to make YOLO prediction more accurate and faster, including:

**1. BatchNorm helps**: Add _batch norm_ on all the convolutional layers, leading to significant improvement over convergence.

**2. Image resolution matters**: Fine-tuning the base model with _high resolution_ images improves the detection performance.

**3. Convolutional anchor box detection**: Rather than predicts the bounding box position with fully-connected layers over the whole feature map, YOLOv2 uses _convolutional layers_ to predict locations of _anchor boxes_, like in faster R-CNN. The prediction of spatial locations and class probabilities are decoupled. Overall, the change leads to a slight decrease in mAP, but an increase in recall.

**4. K-mean clustering of box dimensions**: Different from faster R-CNN that uses hand-picked sizes of anchor boxes, YOLOv2 runs k-mean clustering on the training data to find good priors on anchor box dimensions. The distance metric is designed to _rely on IoU scores_:

where is a ground truth box candidate and is one of the centroids. The best number of centroids (anchor boxes) can be chosen by the [elbow method](https://en.wikipedia.org/wiki/Elbow_method_(clustering)).

The anchor boxes generated by clustering provide better average IoU conditioned on a fixed number of boxes.

**5. Direct location prediction**: YOLOv2 formulates the bounding box prediction in a way that it would _not diverge_ from the center location too much. If the box location prediction can place the box in any part of the image, like in regional proposal network, the model training could become unstable.

Given the anchor box of size at the grid cell with its top left corner at , the model predicts the offset and the scale, and the corresponding predicted bounding box has center and size . The confidence score is the sigmoid () of another output .

![Image 7](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/yolov2-loc-prediction.png)

YOLOv2 bounding box location prediction. (Image source: [original paper](https://arxiv.org/abs/1612.08242))

**6. Add fine-grained features**: YOLOv2 adds a passthrough layer to bring _fine-grained features_ from an earlier layer to the last output layer. The mechanism of this passthrough layer is similar to _identity mappings in ResNet_ to extract higher-dimensional features from previous layers. This leads to 1% performance increase.

**7. Multi-scale training**: In order to train the model to be robust to input images of different sizes, a _new size_ of input dimension is _randomly sampled_ every 10 batches. Since conv layers of YOLOv2 downsample the input dimension by a factor of 32, the newly sampled size is a multiple of 32.

**8. Light-weighted base model**: To make prediction even faster, YOLOv2 adopts a light-weighted base model, DarkNet-19, which has 19 conv layers and 5 max-pooling layers. The key point is to insert avg poolings and 1x1 conv filters between 3x3 conv layers.

## YOLO9000: Rich Dataset Training

Because drawing bounding boxes on images for object detection is much more expensive than tagging images for classification, the paper proposed a way to combine small object detection dataset with large ImageNet so that the model can be exposed to a much larger number of object categories. The name of YOLO9000 comes from the top 9000 classes in ImageNet. During joint training, if an input image comes from the classification dataset, it only backpropagates the classification loss.

The detection dataset has much fewer and more general labels and, moreover, labels cross multiple datasets are often not mutually exclusive. For example, ImageNet has a label “Persian cat” while in COCO the same image would be labeled as “cat”. Without mutual exclusiveness, it does not make sense to apply softmax over all the classes.

In order to efficiently merge ImageNet labels (1000 classes, fine-grained) with COCO/PASCAL (< 100 classes, coarse-grained), YOLO9000 built a hierarchical tree structure with reference to [WordNet](https://wordnet.princeton.edu/) so that general labels are closer to the root and the fine-grained class labels are leaves. In this way, “cat” is the parent node of “Persian cat”.

![Image 8](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/word-tree.png)

The WordTree hierarchy merges labels from COCO and ImageNet. Blue nodes are COCO labels and red nodes are ImageNet labels. (Image source: [original paper](https://arxiv.org/abs/1612.08242))

To predict the probability of a class node, we can follow the path from the node to the root:

```
Pr("persian cat" | contain a "physical object") 
= Pr("persian cat" | "cat") 
  Pr("cat" | "animal") 
  Pr("animal" | "physical object") 
  Pr(contain a "physical object")    # confidence score.
```

Note that `Pr(contain a "physical object")` is the confidence score, predicted separately in the bounding box detection pipeline. The path of conditional probability prediction can stop at any step, depending on which labels are available.

## RetinaNet

The **RetinaNet** ([Lin et al., 2018](https://arxiv.org/abs/1708.02002)) is a one-stage dense object detector. Two crucial building blocks are _featurized image pyramid_ and the use of _focal loss_.

## Focal Loss

One issue for object detection model training is an extreme imbalance between background that contains no object and foreground that holds objects of interests. **Focal loss** is designed to assign more weights on hard, easily misclassified examples (i.e. background with noisy texture or partial object) and to down-weight easy examples (i.e. obviously empty background).

Starting with a normal cross entropy loss for binary classification,

where is a ground truth binary label, indicating whether a bounding box contains a object, and is the predicted probability of objectiveness (aka confidence score).

For notational convenience,

Easily classified examples with large , that is, when is very close to 0 (when y=0) or 1 (when y=1), can incur a loss with non-trivial magnitude. Focal loss explicitly adds a weighting factor to each term in cross entropy so that the weight is small when is large and therefore easy examples are down-weighted.

![Image 9](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/focal-loss.png)

The focal loss focuses less on easy examples with a factor of . (Image source: [original paper](https://arxiv.org/abs/1708.02002))

For a better control of the shape of the weighting function (see Fig. 10.), RetinaNet uses an -balanced variant of the focal loss, where works the best.

![Image 10](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/focal-loss-weights.png)

The plot of focal loss weights as a function of , given different values of and .

## Featurized Image Pyramid

The **featurized image pyramid** ([Lin et al., 2017](https://arxiv.org/abs/1612.03144)) is the backbone network for RetinaNet. Following the same approach by [image pyramid](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/#image-pyramid) in SSD, featurized image pyramids provide a basic vision component for object detection at different scales.

The key idea of feature pyramid network is demonstrated in The base structure contains a sequence of _pyramid levels_, each corresponding to one network _stage_. One stage contains multiple convolutional layers of the same size and the stage sizes are scaled down by a factor of 2. Let’s denote the last layer of the -th stage as .

![Image 11](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/featurized-image-pyramid.png)

The illustration of the featurized image pyramid module. (Replot based on figure 3 in [FPN paper](https://arxiv.org/abs/1612.03144))

Two pathways connect conv layers:

*   **Bottom-up pathway** is the normal feedforward computation.
*   **Top-down pathway** goes in the inverse direction, adding coarse but semantically stronger feature maps back into the previous pyramid levels of a larger size via lateral connections. 
    *   First, the higher-level features are upsampled spatially coarser to be 2x larger. For image upscaling, the paper used nearest neighbor upsampling. While there are many [image upscaling algorithms](https://en.wikipedia.org/wiki/Image_scaling#Algorithms) such as using [deconv](https://www.tensorflow.org/api_docs/python/tf/layers/conv2d_transpose), adopting another image scaling method might or might not improve the performance of RetinaNet.
    *   The larger feature map undergoes a 1x1 conv layer to reduce the channel dimension.
    *   Finally, these two feature maps are merged by element-wise addition. 
The lateral connections only happen at the last layer in stages, denoted as

, and the process continues until the finest (largest) merged feature map is generated. The prediction is made out of every merged map after a 3x3 conv layer, .

According to ablation studies, the importance rank of components of the featurized image pyramid design is as follows: **1x1 lateral connection**> detect object across multiple layers > top-down enrichment > pyramid representation (compared to only check the finest layer).

## Model Architecture

The featurized pyramid is constructed on top of the ResNet architecture. Recall that [ResNet](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/TBA) has 5 conv blocks (= network stages / pyramid levels). The last layer of the -th pyramid level, , has resolution lower than the raw input dimension.

RetinaNet utilizes feature pyramid levels to :

*    to are computed from the corresponding ResNet residual stage from to . They are connected by both top-down and bottom-up pathways.
*    is obtained via a 3×3 stride-2 conv on top of 
*    applies ReLU and a 3×3 stride-2 conv on .

Adding higher pyramid levels on ResNet improves the performance for detecting large objects.

Same as in SSD, detection happens in all pyramid levels by making a prediction out of every merged feature map. Because predictions share the same classifier and the box regressor, they are all formed to have the same channel dimension d=256.

There are A=9 anchor boxes per level:

*   The base size corresponds to areas of to pixels on to respectively. There are three size ratios, .
*   For each size, there are three aspect ratios {1/2, 1, 2}.

As usual, for each anchor box, the model outputs a class probability for each of classes in the classification subnet and regresses the offset from this anchor box to the nearest ground truth object in the box regression subnet. The classification subnet adopts the focal loss introduced above.

![Image 12](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/retina-net.png)

The RetinaNet model architecture uses a [FPN](https://arxiv.org/abs/1612.03144) backbone on top of ResNet. (Image source: the [FPN](https://arxiv.org/abs/1612.03144) paper)

## YOLOv3

[YOLOv3](https://pjreddie.com/media/files/papers/YOLOv3.pdf) is created by applying a bunch of design tricks on YOLOv2. The changes are inspired by recent advances in the object detection world.

Here are a list of changes:

**1. Logistic regression for confidence scores**: YOLOv3 predicts an confidence score for each bounding box using _logistic regression_, while YOLO and YOLOv2 uses sum of squared errors for classification terms (see the [loss function](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/#loss-function) above). Linear regression of offset prediction leads to a decrease in mAP.

**2. No more softmax for class prediction**: When predicting class confidence, YOLOv3 uses _multiple independent logistic classifier_ for each class rather than one softmax layer. This is very helpful especially considering that one image might have multiple labels and not all the labels are guaranteed to be mutually exclusive.

**3. Darknet + ResNet as the base model**: The new Darknet-53 still relies on successive 3x3 and 1x1 conv layers, just like the original dark net architecture, but has residual blocks added.

**4. Multi-scale prediction**: Inspired by image pyramid, YOLOv3 adds several conv layers after the base feature extractor model and makes prediction at three different scales among these conv layers. In this way, it has to deal with many more bounding box candidates of various sizes overall.

**5. Skip-layer concatenation**: YOLOv3 also adds cross-layer connections between two prediction layers (except for the output layer) and earlier finer-grained feature maps. The model first up-samples the coarse feature maps and then merges it with the previous features by concatenation. The combination with finer-grained information makes it better at detecting small objects.

Interestingly, focal loss does not help YOLOv3, potentially it might be due to the usage of and — they increase the loss from bounding box location predictions and decrease the loss from confidence predictions for background boxes.

Overall YOLOv3 performs better and faster than SSD, and worse than RetinaNet but 3.8x faster.

![Image 13](https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/yolov3-perf.png)

The comparison of various fast object detection models on speed and mAP performance. (Image source: [focal loss](https://arxiv.org/abs/1708.02002) paper with additional labels from the [YOLOv3](https://pjreddie.com/media/files/papers/YOLOv3.pdf) paper.)

* * *

Cited as:

```
@article{weng2018detection4,
  title   = "Object Detection Part 4: Fast Detection Models",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2018",
  url     = "https://lilianweng.github.io/posts/2018-12-27-object-recognition-part-4/"
}
```

## Reference

[1] Joseph Redmon, et al. [“You only look once: Unified, real-time object detection.”](https://www.cv-foundation.org/openaccess/content_cvpr_2016/papers/Redmon_You_Only_Look_CVPR_2016_paper.pdf) CVPR 2016.

[2] Joseph Redmon and Ali Farhadi. [“YOLO9000: Better, Faster, Stronger.”](http://openaccess.thecvf.com/content_cvpr_2017/papers/Redmon_YOLO9000_Better_Faster_CVPR_2017_paper.pdf) CVPR 2017.

[3] Joseph Redmon, Ali Farhadi. [“YOLOv3: An incremental improvement.”](https://pjreddie.com/media/files/papers/YOLOv3.pdf).

[4] Wei Liu et al. [“SSD: Single Shot MultiBox Detector.”](https://arxiv.org/abs/1512.02325) ECCV 2016.

[5] Tsung-Yi Lin, et al. [“Feature Pyramid Networks for Object Detection.”](https://arxiv.org/abs/1612.03144) CVPR 2017.

[6] Tsung-Yi Lin, et al. [“Focal Loss for Dense Object Detection.”](https://arxiv.org/abs/1708.02002) IEEE transactions on pattern analysis and machine intelligence, 2018.

[7] [“What’s new in YOLO v3?”](https://towardsdatascience.com/yolo-v3-object-detection-53fb7d3bfe6b) by Ayoosh Kathuria on “Towards Data Science”, Apr 23, 2018.