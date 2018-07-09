# Image quality assessment
This repository provides an implementation of an aesthetic and technical image quality model based on Google's research paper ["NIMA: Neural Image Assessment"](https://arxiv.org/pdf/1709.05424.pdf). You can find a quick introduction on their [Research Blog](https://research.googleblog.com/2017/12/introducing-nima-neural-image-assessment.html).

NIMA consists of two models that aim to predict the aesthetic and technical quality of images, respectively. The models are trained via transfer learning, where ImageNet pre-trained CNNs are used and fine-tuned for the classification task.

The provided code allows to use any of the pre-trained models in [Keras](https://keras.io/applications/). We further provide Docker images for local CPU training and remote GPU training on AWS EC2, as well as pre-trained models on the [AVA](https://github.com/ylogx/aesthetics/tree/master/data/ava) and [TID2013](http://www.ponomarenko.info/tid2013.htm) datasets.

We welcome all kinds of contributions, especially new model architectures and/or hyperparameter combinations that improve the performance of the currently published models (see [Contribute](#contribute)).


## Trained models
| <sub>Predictions from aesthetic model</sub>
| :--:
| ![](/_readme/images_aesthetic/aesthetic1.jpg_aesthetic.svg)


| <sub>Predictions from technical model</sub>
| :--:
| ![](/_readme/images_technical/techncial3.jpgtechnical.svg)




We provide trained models, for both aesthetic and technical classifications, that use MobileNet as the base CNN. The models and their respective config files are stored under `models/MobileNet`. They achieve the following performance

Model      | Dataset | EMD  | LCC | SRCC  
----- |  ------- | ---  | --- | ----  
MobileNet aesthetic | AVA | 0.071 |0.626|0.609
MobileNet technical | TID2013 | 0.107 |0.652|0.675



## Getting started

1. Install [Docker](https://docs.docker.com/install/)

2. Build docker image `docker build -t nima-cpu . -f Dockerfile.cpu`

In order to train remotely on **AWS EC2**

3. Install [Docker Machine](https://docs.docker.com/machine/install-machine/)

4. Install [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)


## Predict
In order to run predictions on an image or batch of images you can run the prediction script

1. Single image file
```
./predict  \
--docker-image nima-cpu \
--base-model-name MobileNet \
--weights-file $(pwd)/models/MobileNet/weights_mobilenet_technical_0.11.hdf5 \
--image-source $(pwd)/src/tests/test_images/42039.jpg
```

2. All image files in a directory
```
./predict  \
--docker-image nima-cpu \
--base-model-name MobileNet \
--weights-file $(pwd)/models/MobileNet/weights_mobilenet_technical_0.11.hdf5 \
--image-source $(pwd)/src/tests/test_images
```


## Train locally on CPU

1. Download dataset (see instructions under [Datasets](#datasets))

2. Run the local training script (e.g. for TID2013 dataset)
```
./train-local \
--config-file $(pwd)/models/MobileNet/config_mobilenet_technical.json \
--samples-file $(pwd)/data/TID2013/tid_labels_train.json \
--image-dir /path/to/image/dir/local
```
This will start a training container from the Docker image `nima-cpu` and create a timestamp train job folder under `train_jobs`, where the trained model weights and logs will be stored. The `--image-dir` argument requires the path of the image directory on your local machine.

  In order to stop the last launched container run
```
CONTAINER_ID=$(docker ps -l -q)
docker container stop $CONTAINER_ID
```

  In order to stream logs from last launched container run
```
CONTAINER_ID=$(docker ps -l -q)
docker logs $CONTAINER_ID --follow
```

## Train remotely on AWS EC2

1. Configure your AWS CLI. Ensure that your account has limits for GPU instances and read/write access to the S3 bucket specified in config file [[link](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html)]
```
aws configure
```

2. Launch EC2 instance with Docker Machine. Choose an Ubuntu AMI based on your region (https://cloud-images.ubuntu.com/locator/ec2/).
For example, to launch a `p2.xlarge` EC2 instance named `ec2-p2` run
(NB: change region, VPC ID and AMI ID as per your setup)
```
docker-machine create --driver amazonec2 \
                      --amazonec2-region eu-west-1 \
                      --amazonec2-ami ami-58d7e821 \
                      --amazonec2-instance-type p2.xlarge \
                      --amazonec2-vpc-id vpc-abc \
                      ec2-p2
```

3. Download dataset to EC2 instance (see instructions under [Datasets](#datasets)). We recommend to save the AMI with the downloaded data for future use.


4. Run the remote EC2 training script (e.g. for AVA dataset)
```
./train-ec2 \
--docker-machine ec2-p2 \
--config-file $(pwd)/models/MobileNet/config_mobilenet_aesthetic.json \
--samples-file $(pwd)/data/AVA/ava_labels_train.json \
--image-dir /path/to/image/dir/remote
```
The training progress will be streamed to your terminal. After the training has finished, the train outputs (logs and best model weights) will be stored on S3 in a timestamped folder. The S3 output bucket can be specified in the **config file**. The `--image-dir` argument requires the path of the image directory on your remote instance.


## Contribute
We welcome all kinds of contributions and will publish the performances from new models in the performance table under [Trained models](#trained-models).

For example, to train a new aesthetic NIMA model based on InceptionV3 ImageNet weights, you just have to change the `base_model_name` parameter in the config file `models/MobileNet/config_mobilenet_aesthetic.json` to "InceptionV3". You can also control all major hyperparameters in the config file, like learning rate, batch size, or dropout rate.


## Datasets
This project uses two datasets to train the NIMA model:
1. [**AVA**](https://github.com/ylogx/aesthetics/tree/master/data/ava) used for aesthetic rating
2. [**TID2013**](http://www.ponomarenko.info/tid2013.htm) used for technical rating

This [discussion](https://www.reddit.com/comments/5sa3ag) provides further details on how to download the AVA dataset. For training on AWS EC2 we recommend to build a custom AMI with the AVA images stored on it. This has proven much more viable than copying the entire dataset from S3 to the instance for each training job.


## Label files
The **train** script requires JSON label files in the format
```
[
  {
    "image_id": "231893",
    "label": [2,8,19,36,76,52,16,9,3,2]
  },
  {
    "image_id": "746672",
    "label": [1,2,7,20,38,52,20,11,1,3]
  },
  ...
]
```

The label for each image is the normalized or un-normalized frequency distribution of ratings 1-10.

For the AVA dataset these frequency distributions are given in the raw data files. For the TID2013 dataset we inferred the normalized frequency distribution, i.e. probability distribution, by finding the maximum entropy distribution that satisfies the mean score. The code to generate the TID2013 labels can be found under `data/TID2013/get_labels.py`.

For both datasets we provide train and test set label files stored under
```
data/AVA/ava_labels_train.json
data/AVA/ava_labels_test.json
```
and
```
data/TID2013/tid2013_labels_train.json
data/TID2013/tid2013_labels_test.json
```

For the AVA dataset we randomly assigned 90% of samples to the train set, and 10% to the test set, and throughout training a 5% validation set will be split from the training set to evaluate the training performance after each epoch. For the TID2013 dataset we split the train/test sets by reference images, to ensure that no reference image, and any of its distortions, enters both the train and test set.

## Maintainers
* Christopher Lennan, github: [clennan](https://github.com/clennan)
* Hao Nguyen, github: [MrBanhBao](https://github.com/MrBanhBao)


## Copyright

See [LICENSE](LICENSE) for details.
