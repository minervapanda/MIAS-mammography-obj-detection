# Breast cancer tumour detection on mammograms

## Requirements

- Ubuntu 18.04
- GCC >= 4.9
- CUDA 10.1 & cuDNN 7.6
- Anaconda 3 ([install. instructions](https://problemsolvingwithpython.com/01-Orientation/01.05-Installing-Anaconda-on-Linux/))

## References

- Faster-RCNN paper: [arxiv.org/pdf/1506.01497.pdf](https://arxiv.org/pdf/1506.01497.pdf)
- RetinaNet paper: [arxiv.org/pdf/1708.02002.pdf](https://arxiv.org/pdf/1708.02002.pdf)
- FCOS paper: [arxiv.org/pdf/1904.01355.pdf](https://arxiv.org/pdf/1904.01355.pdf)
- Faster-RCNN & RetinaNet implementations (Detectron2): [github.com/facebookresearch/detectron2](https://github.com/facebookresearch/detectron2)
- FCOS implem. repo.: [github.com/tianzhi0549/FCOS](https://github.com/tianzhi0549/FCOS)

## Installation instructions

Start by cloning this repo:

```
git clone https://github.com/delmalih/MIAS-mammography-obj-detection
```

### 1. Faster R-CNN instructions

- First, create an environment :

```
conda create --name faster-r-cnn
conda activate faster-r-cnn
conda install ipython pip
cd MIAS-mammography-obj-detection
pip install -r requirements.txt
cd ..
```

- Then, run these commands :

```
# install pytorch & other packages
conda install pytorch torchvision cudatoolkit=10.1 -c pytorch
pip install cython
pip install 'git+https://github.com/cocodataset/cocoapi.git#subdirectory=PythonAPI'

# build detectron2
git clone https://github.com/facebookresearch/detectron2
cd detectron2
python setup.py build develop
cd ..
```

### 2. RetinaNet instructions

- First, create an environment :

```
conda create --name retinanet python=3.6
conda activate retinanet
conda install ipython pip
cd MIAS-mammography-obj-detection
pip install -r requirements.txt
cd ..
pip install tensorflow-gpu==1.9
pip install keras==2.2.5
```

- Then, run these commands :

```
# clone keras-retinanet repo
git clone https://github.com/fizyr/keras-retinanet
cd keras-retinanet
pip install .
python setup.py build_ext --inplace
```

- Finally, replace the `keras_retinanet/preprocessing/coco.py` file by [this file](https://github.com/delmalih/mias-mammography-obj-detection/blob/master/retinanet/coco.py)

### 3. FCOS instructions

- First, create an environment :

```
conda create --name fcos
conda activate fcos
conda install ipython pip
cd MIAS-mammography-obj-detection
pip install -r requirements.txt
cd ..
```

- Then, follow [these instructions](https://github.com/tianzhi0549/FCOS#installation)

## How it works

### 1. Download the MIAS Database

Run these commands to download to MIAS database :

```
mkdir mias-db && cd mias-db
wget http://peipa.essex.ac.uk/pix/mias/all-mias.tar.gz
tar -zxvf all-mias.tar.gz
rm all-mias.tar.gz && cd ..
```

And replace the `mias-db/Info.txt` by [this one](https://raw.githubusercontent.com/delmalih/MIAS-mammography-obj-detection/master/utils/Info.txt)

### 2. Generate COCO or VOC augmented data

It is possible to generate COCO or VOC annotations from raw data (`all-mias` folder + `Info.txt` annotations file) through 2 scripts: `generate_{COCO|VOC}_annotations.py` :

```
python generate_{COCO|VOC}_annotations.py --images (or -i) <Path to the images folder> \
                                          --annotations (or -a) <Path to the .txt annotations file> \
                                          --output (or -o) <Path to output folder> \
                                          --aug_fact <Data augmentation factor> \
                                          --train_val_split <Percetange of the train folder (default 0.9)>
```

For example, to generate 10x augmented COCO annotations, run this command :

```
python generate_COCO_annotations.py --images ../mias-db/ \
                                    --annotations ../mias-db/Info.txt \
                                    --output ../mias-db/COCO \
                                    --aug_fact 20 \
                                    --train_val_split 0.9
```

### 3. How to run a training

#### 3.1 Faster R-CNN

To run a training with the Faster-RCNN:

- Go to the faster-r-cnn directory: `cd faster-r-cnn`
- Change conda env: `conda deactivate && conda activate faster-r-cnn`
- Download the [Resnet_101_FPN model](https://download.pytorch.org/models/maskrcnn/e2e_faster_rcnn_R_101_FPN_1x.pth)
- Trim the model: `python trim_detectron_model.py --pretrained_path e2e_faster_rcnn_R_101_FPN_1x.pth --save_path base_model.pth`
- Edit the `maskrcnn-benchmark/maskrcnn_benchmark/config/paths_catalog.py` file and put these lines in the `DATASETS` dictionary :
```
  DATASETS = {
    ...,
    "mias_train_cocostyle": {
        "img_dir": "<PATH_TO_'mias-db'_folder>/<COCO_FOLDER>/images/train",
        "ann_file": "<PATH_TO_'mias-db'_folder>/<COCO_FOLDER>/annotations/instances_train.json"
    },
    "mias_val_cocostyle": {
        "img_dir": "<PATH_TO_'mias-db'_folder>/<COCO_FOLDER>/images/val",
        "ann_file": "<PATH_TO_'mias-db'_folder>/<COCO_FOLDER>/annotations/instances_val.json"
    },
  }
```
- In the `maskrcnn-benchmark/maskrcnn_benchmark/data/datasets/coco.py`, comment line 84 to 92 :
```
    # if anno and "segmentation" in anno[0]:
    #     masks = [obj["segmentation"] for obj in anno]
    #     masks = SegmentationMask(masks, img.size, mode='poly')
    #     target.add_field("masks", masks)

    # if anno and "keypoints" in anno[0]:
    #     keypoints = [obj["keypoints"] for obj in anno]
    #     keypoints = PersonKeypoints(keypoints, img.size)
    #     target.add_field("keypoints", keypoints)
```
- Run this command :
```
python train.py --config-file mias_config.yml
```

#### 3.2 RetinaNet

To run a training with the retinanet :

```
cd retinanet
conda deactivate && conda activate retinanet
python train.py --compute-val-loss \ # Computer val loss or not
                --tensorboard-dir <Path to the tensorboard directory> \
                --batch-size <Batch size> \
                --epochs <Nb of epochs> \
                coco <Path to the COCO dataset>
```

And if you want to see the tensorboard, run on another window :

```
tensorboard --logdir <Path to the tensorboard directory>
```

#### 3.3 FCOS

To run a training with the FCOS Object Detector :

- Follow [these instructions](https://github.com/tianzhi0549/FCOS/issues/54#issuecomment-497558687)
- Run this command :

```
cd fcos
conda deactivate && conda activate fcos
python train.py --config-file <Path to the config file> \
                OUTPUT_DIR <Path to the output dir for the logs>
```

### 4. How to run an inference

#### 4.1 Faster R-CNN

To run an inference, you need a pre-trained model. Run this command:

```
cd faster-r-cnn
conda deactivate && conda activate faster-r-cnn
python inference.py --config-file <Path to the config file> \
                    MODEL.WEIGHT <Path to weights of the model to load> \
                    TEST.IMS_PER_BATCH <Nb of images per batch>
```


#### 4.2 RetinaNet

- Put the images you want to run an inference on, in `<Name of COCO dataset>/<Name of folder>`
- Run this command :

```
cd retinanet
conda deactivate && conda activate retinanet
python inference.py --snapshot <Path of the model snapshot> \
                    --set_name <Name of the inference folder in the COCO dataset> \
                    coco <Path to the COCO dataset>
```

#### 4.3 FCOS

```
cd fcos
conda deactivate && conda activate fcos
python inference.py --config-file <Path to the config file> \
                    MODEL.WEIGHT <Path to weights of the model to load> \
                    TEST.IMS_PER_BATCH <Nb of images per batch>
```

## Results

|Metric|Faster-RCNN|RetinaNet|FCOS|
|------|-----------|---------|----|
|mAP   |98,70%     |94,97%   |98,20%|
|Precision|94,12%  |100,00%  |94,44%|
|Recall|98,65%     |94,72%   |98,20%|
|F1-score|96,22%   |96,93%   |96,25%|


## Poster : Breast Cancer Detection Contest

<img width="892" alt="Capture d’écran 2019-11-09 à 12 25 53" src="https://user-images.githubusercontent.com/54987951/68527857-1d940a00-02ec-11ea-8730-9b1355024239.png">
