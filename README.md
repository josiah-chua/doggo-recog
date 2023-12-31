# Overview
This personal MLOps project is to implement an end-to-end pipeline to create a telegram bot that uses deep learning to differentiate dog breeds.
The main objectives are to test the effectiveness of Neptune AI, an ML model monitoring and storage system, and familiarise myself with PyTorch Lightning a Pytorch wrapper package that allows for faster implementation of training deep learning models.

## MLOps Pipeline
![image](https://github.com/josiah-chua/doggo-recog/assets/81459293/8d391a96-ec36-4b51-b1c9-b8c160c4ade7)


## File structure
```
doggo-recog/
├── credentials/
├── data/
│   ├── deployment_model/
│   ├── model_data/
│   ├── new_data/
│   └── temp_data/
├── src/
│   ├── a_preprocessing.py
│   ├── b_experiment.py
│   ├── c_training.py
│   ├── d_api.py
│   └── utils/
│       ├── utils_data.py
│       ├── utils_envvar.py
│       ├── utils_gcs.py
│       ├── utils_model.py
│       ├── utils_preprocessing.py
│       └── utils_training.py
├── .env
├── venv
├── .gitignore
├── README.md
└── .requirements.txt
```

## Original Dataset
The dataset can be found in the dataset branch. It is from the Standford dogs dataset but the filename has been cleaned up to include the label and a unique id (based on datetime ms) for each photo and has been converted to jpg.
A similar naming convention will also be used in future implementation of data collection from users.
The dataset has been cleaned beforehand and formatted correctly into its 3 channels (RGB) as a couple of photos were noted to have 4 channels (png photos contain a 4th opacity channel).

![image](https://github.com/josiah-chua/doggo-recog/assets/81459293/12ce3fed-7e08-4f2a-a8df-e8be1476350e)


## Preprocessing
Since the current dataset is quite clean, the preprocessing script currently only has a placeholder preprocessing function. However should user data be stored, there might be a need for further preprocessing which will be added to the function. This could be a future enhancement


After the preprocessing, the script generates npy array of the photos and stores the numpy array files in the GCS preprocessed bucket and the raw images in the raw image bucket. There should be 4 different GCS buckets, new-raw-images, new-preprocessed-images, raw-images, preprocessed-images. In this first iteration, the buckets with new will not be used as they are used for storing new user data for retraining, and the files should be stored in the other 2 buckets. 


There are 2 make executors for the preprocessing stage:


```make preprocess_training_files```: will store the preprocessed files into  raw-images, preprocessed-images buckets


```make preprocess_new_files``` : will store the preprocessed files into new-preprocessed-images, raw-images buckets

## Experimentation
The experiment and training are similar, however, the experimentation section is used because deep learning models take a long time to train. Hence to evaluate the effectiveness of certain models and certain hyperparameters, we can do smaller experiments with fewer classes and lesser data per class to have a rough gauge of the performance. While having a better performance in smaller datasets will not guarantee better results in the full dataset, it will be able to give us a sense of how effective it is by looking at the training metrics and prior knowledge of the model architecture.

This is where Pytorch Lightning and Neptune.ai come in. Pytorch lightning allows us to package our models nicely and easily attach loggers and callbacks for to monitor our training progress. Training metrics can then be sent to the Neptune server through the NeptuneLogger and visualized on a dashboard.


![image](https://github.com/josiah-chua/doggo-recog/assets/81459293/d0004fca-7564-4f14-ae90-8641a3a6865b)


For the experimentation stage, one can experiment with different model architectures (vgg, efficientnet, resnet) and these have been initiated with pre-trained ImageNet Weights(IMAGENET1K_V1) for [transfer learning](https://machinelearningmastery.com/transfer-learning-for-deep-learning/) to reduce the training time needed.

The other hyperparameters to try are:
- image input size (default 224)
- batch size (default 32)
- learning rate (default 1e-3)
- transformation probability (default = 0.025): Chance that the training image will go through a random [augmentation](https://pytorch.org/vision/stable/transforms.html)

These hyperparameters will also be stored in Neptune in the corresponding run.

## Training
Before training, the images are converted into pytorch datasets, with transform functions such as, change contrast, perspective, brightness blur etc. and these augmentations will occur randomly as the dataset is loaded every epoch, allowing the model to generalize features better and become more robust. [To understand it better](https://discuss.pytorch.org/t/data-augmentation-in-pytorch/7925?u=nikronic). Hence there is not much need to generate transformed data in the preprocessing stage as new data although you can still do so if you want.

The training process is handled by pytorch lightning trainer and the callbacks used are early stopping and checkpointing. The patience and max epochs can both be set for the training and early stopping. The checkpointing will only checkpoint the last training weights and the weights of the best-performing epoch (this will only be saved for the training stage). The model used MulticlassROCAUC for evaluation.

In the training stage, deployment artifacts will also be stored in Neptune's model registry. The best checkpoint (for retaining) along with its [torchscript model](https://towardsdatascience.com/pytorch-jit-and-torchscript-c2a77bac0fff) file(for deployment) and the label dict to convert the labels to the corresponding dog breeds, will be saved to the model registry.

In the model registry, deployment of different models is straightforward as it will just require changing that model stage to production. When deployed the API will search for the models in production and take the first model to be used.

![image](https://github.com/josiah-chua/doggo-recog/assets/81459293/f8173fad-28c2-404d-9281-44216994982f)

# Usage

## Data
The data can be downloaded from the data branch and the contents should be placed into data/model_data. Then run make ```preprocess_training_files``` to preprocess the files.

Also note all data folders have been git ignored so you will have to make them. When terraform code is done it will create the folders but for now too bad.

## Set up GCP storage bucket

Create the 4 Google Cloud Storage Buckets with the same name defined in your environment variables. Terraform code to deploy infra will be added soon

## Environment Variable
There is a sample environment variable file where you can add the environment variables. But please rename it to .env. 
The Google Cloud Service Account should be [base64 encoded](https://www.base64encode.org/) to a string and placed as an environment variable.

## deployment

To deploy this telegram bot you will first have to follow the first few steps of this [tutorial](https://www.pragnakalp.com/create-telegram-bot-using-python-tutorial-with-examples/) till you generate the telegram bot token to put in your environment variable.

Then set up ngrok as shown [here](https://www.sitepoint.com/use-ngrok-test-local-site/) and create a tunnel to port 8000 using the CLI ```ngrok http 8000```.

Then run the make command ```make launch_bot``` and it should work.

# Future Improvements
:green_circle: - Done, :yellow_circle: In Progress, :red_circle: - Yet To Implement

:yellow_circle: Retraining Script

:yellow_circle: Cloud deployment on GCP Kubecluster with docker images

:red_circle: Training on Cloud Compute

:red_circle: Data Collection from users, feedback, and retraining (depending on the data collection policies)

# Relevant Reading Materials
[Neptune overview](https://neptune.ai/):

[Pytorch lightning overview](https://lightning.ai/docs/pytorch/stable/notebooks/lightning_examples/mnist-hello-world.html):
