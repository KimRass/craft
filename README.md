# What is 'CRAFT'?
- [CRAFT: Character-Region Awareness For Text detection](https://github.com/clovaai/CRAFT-pytorch)

# Directory Structure
```
CRAFT-pytorch
├── LICENSE (x)
├── README.md (x)
├── basenet
│   ├── __init__.py (x)
│   └── vgg16_bn.py (o)
├── craft.py (o)
├── craft_utils.py (o)
├── figures
│   └── craft_example.gif (x)
├── file_utils.py (x)
├── imgproc.py (o)
├── refinenet.py (o)
├── requirements.txt (x)
└── test.py (o)
```

# Model Architecture
- Model architecture
- <img src="https://miro.medium.com/max/1400/1*b6I-Bdj5itX7tllJ5HRKbg.png" width="400">

# Paper Summary
- [Character Region Awareness for Text Detection](https://arxiv.org/pdf/1904.01941.pdf)
- Reference: https://medium.com/@msmapark2/character-region-awareness-for-text-detection-craft-paper-%EB%B6%84%EC%84%9D-da987b32609c
## Character-level Awareness
  - These methods mainly train their networks to localize wordlevel bounding boxes. However, they may suffer in difficult cases, such as texts that are curved, deformed, or extremely long, which are hard to detect with a single bounding box.
  - Alternatively, character-level awareness has many advantages when handling challenging texts by linking the successive characters in a bottom-up manner.
  - In this paper, we propose a novel text detector that localizes the individual character regions and links the detected characters to a text instance.
  - Most methods detect text with words as its unit, but defining the extents to a word for detection is non-trivial since words can be separated by various criteria, such as meaning, spaces or color. In addition, the boundary of the word segmentation cannot be strictly defined, so the word segment itself has no distinct semantic meaning. This ambiguity in the word annotation dilutes the meaning of the ground truth for both regression and segmentation approaches.
## Training
- Unfortunately, most of the existing text datasets do not provide characterlevel annotations, and the work needed to obtain character level ground truths is too costly.
- The training procedure includes two steps: we first use the SynthText dataset [6] to train the network for 50k iterations, then each benchmark dataset is adopted to fine-tune the model. 
- During fine-tuning, the SynthText dataset is also used at a rate of 1:5 to make sure that the character regions are surely separated.
- On-line Hard Negative Mining
  - In order to filter out texture-like texts in natural scenes, On-line Hard Negative Mining [33] is applied at a ratio of 1:3.
- Data augmentation
  - Also, basic data augmentation techniques like crops, rotations, and/or color variations are applied.
### Weakly-supervised Learning
- Weakly-supervised training requires two types of data; quadrilateral annotations for cropping word images and transcriptions for calculating word length. The datasets meeting these conditions are IC13, IC15, and IC17. Other datasets such as MSRA-TD500, TotalText, and CTW-1500 do not meet the requirements.
- Fig. 4
  - <img src="https://miro.medium.com/v2/resize:fit:4800/format:webp/1*vznvf3ewJtZY83roODyy1Q.png" width="600">
- Unlike synthetic datasets, real images in a dataset usually have word-level annotations. Here, we generate character boxes from each word-level annotation in a weakly-supervised manner, as summarized in Fig. 4. **When a real image with word-level annotations is provided, the learned interim model predicts the character region score of the cropped word images to generate character-level bounding boxes. In order to reflect the reliability of the interim model’s prediction, the value of the confidence map over each word box is computed proportional to the number of the detected characters divided by the number of the ground truth characters, which is used for the learning weight during training.**
- Fig. 6
  - <img src="https://miro.medium.com/v2/resize:fit:4800/format:webp/1*X97KqXolnfz4S2abu10sDw.png" width="600">
- **First, the word-level images are cropped from the original image. Second, the model trained up to date predicts the region score. Third, the watershed algorithm is used to split the character regions, which is used to make the character bounding boxes covering regions. Finally, the coordinates of the character boxes are transformed back into the original image coordinates using the inverse transform from the cropping step. The pseudo-ground truths (pseudo- GTs) for the region score and the affinity score can be generated by the steps described in Fig. 3 using the obtained quadrilateral character-level bounding boxes.**
- When the model is trained using weak-supervision, we are compelled to train with incomplete pseudo-GTs. **If the model is trained with inaccurate region scores, the output might be blurred within character regions. To prevent this, we measure the quality of each pseudo-GTs generated by the model.** In most datasets, the transcription of words is provided and the length of the words can be used to evaluate the confidence of the pseudo-GTs. For a word-level annotated sample $w$ of the training data, let $R(w)$ and $l(w)$ be the bounding box region and the word length of the sample $w$, respectively. Through the charac- ter splitting process, we can obtain the estimated character bounding boxes and their corresponding length of characters $l^{c}(w)$. Then the confidence score $s_{conf}(w)$ for the sample $w$ is computed as,
$$s_{conf}(w) = \frac{l(w) - min\big( l(w),\ \lvert l(w) - l^{c}(w) \rvert \big)}{l(w)}$$
- Comment:
$$If\ l^{c}(w) \le l(w),\ then\ s_{conf}(w) = \frac{l^{c}(w)}{l(w)}$$
$$If\ l(w) \lt l^{c}(w) \le 2l(w),\ then\ s_{conf}(w) = \frac{2l(w) - l^{c}(w)}{l(w)}$$
$$If\ l^{c}(w) \gt 2l(w), then\ s_{conf}(w) = 0$$
### Ground-truth Generation
- Fig. 3. Ground-truth generation procedure
  - <img src="https://miro.medium.com/v2/resize:fit:4800/format:webp/1*u9A_-mrjJF_yd7EtTEz-dA.png" width="600">
- Since character bounding boxes on an image are generally distorted via perspective projections, we use the following steps to approximate and generate the ground truth for both the region score and the affinity score:
  - Prepare a 2-dimensional isotropic Gaussian map
  - Compute perspective transform between the Gaussian map region and each character box
  - Warp Gaussian map to the box area.
- Affinity score map
  - For the ground truths of the affinity score, the affinity boxes are defined using adjacent character boxes. By drawing diagonal lines to connect opposite corners of each character box, we can generate two triangles – which we will refer to as the upper and lower character triangles. Then, for each adjacent character box pair, an affinity box is generated by setting the centers of the upper and lower triangles as corners of the box.
  - Comment: 논문에는 Triangle center가 정확히 무엇을 의미하는지 나와 있지 않습니다. Triangle center에는 Centroid (무게중심), Incenter (내심), Circumcenter (외심), Orthocenter (수심) 등이 있는데, 아마 Centroid를 사용하는 것으로 보입니다. [NENET: An Edge Learnable Network for Link Prediction in Scene Text](https://arxiv.org/pdf/2005.12147.pdf)에서도 Centroid를 사용한다고 설명합니다.
### Loss
$$L = \sum_{p} { S_{c}(p) \cdot \big( \lVert S_{r}(p) - S_{r}^{\*}(p) \rVert _{2}^{2} + \lVert S_{a}(p) - S_{a}^{\*}(p) \rVert _{2}^{2} \big) }$$
- $S_{c}(p)$: Confidence score at pixel $p$ (When training with synthetic data, we can obtain the real ground truth, so $S_{c}(p) = 1$)
- $S_{r}^{\*}(p)$: (Pseudo) Ground-truth region score at pixel $p$
- $S_{r}(p)$: Predicted region score at pixel $p$
- $S_{a}^{\*}(p)$: (Pseudo) Ground-truth affinity score at pixel $p$
- $S_{a}(p)$: Predicted affinity score at pixel $p$
- Comment: Norm
$$\lVert \textbf{x} \rVert_{p} := \Bigl(\sum \lvert x_{i} \rvert^{p}\Bigr)^{1/p}$$
- 그런데 사실상 Region score map과 Affinity score map은 2차원이므로 $p$에서 스칼라를 가집니다. 따라서 $L$은 그냥 다음과 같습니다.
$$L = \sum_{p} { S_{c}(p) \cdot \Bigl\\{ \big( S_{r}(p) - S_{r}^{\*}(p) \big) ^{2} + \big( S_{a}(p) - S_{a}^{\*}(p) \big) ^{2} \Bigr\\} }$$
### Datasets
- We trained CRAFT only on the ICDAR datasets, and tested on the others without fine-tuning. Two different models are trained with the ICDAR datasets. The first model is trained on IC15 to evaluate IC15 only. The second model is trained on both IC13 and IC17 together, which is used for evaluating the other five datasets. No extra images are used for training. The number of iterations for fine-tuning is set to 25k.
- Comment: 아마도 'fine-tuning'이 아니라 training을 '25k' steps 수행했다는 것으로, 단순 오타가 아닐까 싶습니다.
- ICDAR2013 (IC13):
  - Consisting of high-resolution images, 229 for training and 233 for testing, containing **texts in English. The annotations are at word-level using rectangular boxes.**
- ICDAR2015 (IC15):
  - Consisting of 1,000 training images and 500 testing images, both with **texts in English. The annotations are at the word level using quadrilateral boxes.**
- ICDAR2017 (IC17) (https://zenodo.org/record/835489#.Y_tI_uxBwUE):
  - Contains 7,200 training images, 1,800 validation images, and 9,000 testing images **with texts in 9 languages for multi-lingual scene text detection. Similar to IC15, the text regions in IC17 are also annotated by the 4 vertices of quadrilaterals.**
### Online Hard Example Mining
- In order to filter out texture-like texts in natural scenes, On-line Hard Negative Mining [33] is applied at a ratio of 1:3.
- Basic data augmentation techniques like crops, rotations, and/or color variations are applied.
## Evaluation
### Datasets
- MSRA-TD500 (TD500) (http://www.iapr-tc11.org/mediawiki/index.php/MSRA_Text_Detection_500_Database_(MSRA-TD500)):
  - Contains 500 natural images, which are split into 300 training images and 200 testing images, collected both indoors and outdoors using a pocket camera. The images contain **English and Chinese scripts. Text regions are annotated by rotated rectangles.**
  - **MSRA-TD500 does not provide transcriptions.**
- TotalText (TotalText) ([Total-Text-Dataset](https://github.com/cs-chan/Total-Text-Dataset))
  - Contains 1,255 training and 300 testing images. **It especially provides curved texts, which are annotated by polygons and word-level transcriptions.**
- CTW-1500 (CTW) ([SCUT-CTW1500 Datasets](https://github.com/Yuliang-Liu/Curve-Text-Detector)):
  - Consists of 1,000 training and 500 testing images. Every image has **curved text instances, which are annotated by polygons with 14 vertices.**
## Architecture
- Our framework, referred to as CRAFT for Character Region Awareness For Text detection, is designed with a convolutional neural network producing the character region score and affinity score. The region score is used to localize individual characters in the image, and the affinity score is used to group each character into a single instance.
- The final output has two channels as score maps: the region score and the affinity score.
## Inference
- The final output has two channels as score maps: the region score and the affinity score.
- *At the inference stage, the final output can be delivered in various shapes, such as word boxes or character boxes, and further polygons*.
- Inference
  - <img src="https://miro.medium.com/max/1400/1*_EyygIYQyQPqUk-w-OaKjw.png" width="400">
## Link Refinement
- In the CTW-1500 dataset’s case, two difficult characteristics coexist, namely annotations that are provided at the line-level and are of arbitrary polygons. To aid CRAFT in such cases, a small link refinement network, which we call the LinkRefiner, is used in conjunction with CRAFT.
- *The input of the LinkRefiner is a concatenation of the region score, the affinity score, and the intermediate feature map of CRAFT, and the output is a refined affinity score adjusted for long texts. To combine characters, the refined affinity score is used instead of the original affinity score*, then the polygon generation is performed in the same way as it was performed for TotalText.
- **Only LinkRefiner is trained on the CTW-1500 dataset while freezing CRAFT.** The detailed implementation of LinkRefiner is addressed in the supplementary materials.
- Atrous Spatial Pyramid Pooling (ASPP) in is adopted to ensure a large receptive field for combining distant characters and words onto the same text line.
## References
- [6] [Synthetic Data for Text Localisation in Natural Images](https://arxiv.org/pdf/1604.06646.pdf)

# Region Score Map Difference by Text Colors
- Region score map from original image | Region score map from inverted image
  - <img src="https://i.imgur.com/6EnenSj.jpg" width="800">
- 이미지를 반전한 후에 콤마 (';')를 더 잘 Detect하는 것을 볼 수 있습니다. 아마 학습에 사용된 이미지들에서 하얀색보다는 검은색의 텍스트가 더 많이 존재했기 때문일 것으로 추측됩니다.

# Processing Super High Resolution Images
- 해상도가 매우 큰 이미지는 그 자체로는 메모리 부족 등의 이유로 CRAFT를 사용하여 Infer할 수 없습니다. 따라서 이미지를 적당한 크기로 분할하여 처리해야 합니다.
- 단 이미지를 분할할 때 하나의 문자가 둘 이상으로 나뉘어질 수 있습니다. 이렇게 되면 Text detection이 제대로 수행될 수 없으므로 유의해야 합니다.
- Super high resolution image sample (7,256 x 13,483)
  - <img src="https://i.imgur.com/w0ELNTk.jpg" width="300">
## Algorithm
1. 이미지를 절반의 해상도로 분할하되 다음과 같이 3개로 분리합니다. 이를 통해 하나의 문자가 둘 이상으로 분리되어 Detect되지 못하는 것을 방지할 수 있습니다. 이들 각각을 CRAFT를 사용하여 Infer하고 그 결과들을 합칩니다.
  - Image splitting (Red -> Green -> Blue)
    - <img src="https://i.imgur.com/9Gnmet6.jpg" width="300">
2. 분할된 이미지가 지정된 해상도보다 크다면 각각을 다시 분할합니다. 지정된 해상도 미만이 될 때까지 이것을 반복하여 수행합니다.
- Region score map
  - <img src="https://i.imgur.com/ZbXWURG.jpg" width="300">

# Other Papers
- [Character Region Attention For Text Spotting](https://arxiv.org/pdf/2007.09629.pdf)
