---
title: "Deploy model with FastAPI and Heroku"
date: 2022-03-15T20:40:35+01:00
draft: false
---

In this blog we will see how we can deploy our model with FastAPI and Heroku. Below technologies will be used:

* FastAPI
* Heroku
* Docker
* Github workflow
* AWS

Our model will classify images of dogs and cats.
### First Generating the model

We use the data from https://www.kaggle.com/c/dogs-vs-cats . A model is creted using pretrained resnet model. The kaggle notebook is available [here](https://www.kaggle.com/sabz2301/dog-vs-cat). Not really a SOTA model but it is good enough for our purposes.
We saved the model in AWS S3. We will use the model in our FastAPI server.

### The FastAPI service

Our FastAPI server will be a simple REST API. It has one enpoint `/pet` which will accept an image file and return the prediction. Our code look like this:

```
@app.post("/pet")
async def predict(file: UploadFile=File(...)):
    image = read_imagefile(await file.read())
    tfms= transforms.Compose([
                                            transforms.Resize((224,224)),
                                            transforms.RandomRotation(20),
                                            transforms.RandomVerticalFlip(p=0.1),
                                            transforms.ToTensor(),
                                            transforms.Normalize((0.485, 0.456, 0.406), (0.229, 0.224, 0.225)),
                                            ])
    image = tfms(image)
    image = image[None,:]
    dcModel = DogCatModel(models.resnet34(pretrained=False))
    dcModel.load_state_dict(torch.load('/code/model.pth',map_location=torch.device('cpu')))
    dcModel.eval()
    with torch.no_grad():
        prediction= dcModel(image)
        val,label = torch.max(prediction,dim=1)
        log.info(f"Prediction: {label.item()} with val {val.item()}")
        if val > 0.7:
            label = label.item()
            return {"prediction": label}
        else:
            return {"prediction": "Unknown"}
```
As you can see the code is a pretty straight forward. We read the image file and transform it to a tensor. We load the model and make a prediction.
The complete code can be found [here](https://github.com/njoysubho/fastapi-dog-cat).

Now that we have our model and the FastApi service we can now deploy it.

### The Deployment. 
First an application need to be created on heroku. I named the application `dog-cat-fastapi`.
We can upload the whole code as zip and heroku will automatically deploy the code, however I decided to use docker image as I also wanted to run the code locally.

The dockerFile looks like 
```
FROM ubuntu:20.04

WORKDIR /code

# install utilities
RUN apt-get update && \
    apt-get install --no-install-recommends -y curl && \
    apt-get install -y python3.8 python3-distutils python3-pip python3-apt
# Installing python dependencies
RUN python3 -m pip --no-cache-dir install --upgrade pip && \
    python3 --version && \
    pip3 --version
# pip install aws cli
RUN pip3 install awscli 
ARG MODEL_NAME
ARG AWS_ACCESS_ID
ARG AWS_ACCESS_SECRET
ENV AWS_ACCESS_KEY_ID=$AWS_ACCESS_ID
ENV AWS_SECRET_ACCESS_KEY=$AWS_ACCESS_SECRET
ENV AWS_DEFAULT_REGION=eu-west-1
# Copy model files
RUN aws s3 cp s3://datascience-sab/${MODEL_NAME}.pth .

COPY requirements.txt .

RUN pip3 install --no-cache-dir --upgrade -r /code/requirements.txt

COPY ./app /code/app 

EXPOSE $PORT

CMD gunicorn -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT app.main:app
```
A few things to notice here - 

* `RUN aws s3 cp s3://datascience-sab/${MODEL_NAME}.pth .` command will pull the model file from S3 bucket.
* In order to access aws S3 it needs credential, for this worklfow we are passing the credential from github secret and set those in ENV. 
* When I first deployed the app I could not connect to it , turned out that Heroku does not support the `EXPOSE` command and it sets the random port as env variable named `PORT`an we can use that to expose the port. the belo command will make the app avilable in  the port
`CMD gunicorn -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT app.main:app`

Finally we make our github workflow below is the step to deploy on heroku 

```
....
....
- name: Build and push
      env:
        HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
        AWS_ACCESS_ID: ${{ secrets.AWS_ACCESS_ID }}
        AWS_ACCESS_SECRET: ${{ secrets.AWS_ACCESS_SECRET }}
      run: heroku container:push web --app=${{ secrets.HEROKU_APP_NAME }} --arg AWS_ACCESS_ID=${{ secrets.AWS_ACCESS_ID }},AWS_ACCESS_SECRET=${{ secrets.AWS_ACCESS_SECRET }},MODEL_NAME=model
    - name: Release
      env:
        HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
      run: heroku container:release web --app=${{ secrets.HEROKU_APP_NAME }}  
```
we pass app name, secrets the model name as build args to `heroku container:push` command.
After this we use `heroku container:release` command to release the app.
The complete github workflow can be found [here](https://github.com/njoysubho/fastapi-dog-cat/blob/main/.github/workflows/build-release-deploy.yml).

With this we can deploy our app to heroku. 

Below are some improvements that can be done

* Enhance the model .
* Introduce model versioning. 
* Improved logging, tracing and metrics for the service.
* Also clean up docker image and stream line all RUN and pip commands.

Thats it for now. It was really satisfying for me to learn about the fastapi framework and deploy a model on heroku.