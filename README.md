# Deploy pre-trained model with flask on Docker

> This article shows how to use Flask to deploy a pre-trained model as a web api on docker

## Requirements

Please make sure:

- your device has installed docker and you are familiar with docker
- familiar with Python
- you have a model trained(I use ResNet)
- Azure Data Science Virtual Machine (Optional)

## Write flask server code

```python
@app.route('/upload', methods=['POST'])
def upload_file(): 
    if model ==None:
        init()
    if request.method == 'POST':
        f = request.files['the_file']
        imgPath = 'a.jpg'        
        f.save(imgPath)
        im = image.load_img(imgPath, target_size=(224, 224))
        im = image.img_to_array(im)
        im = np.expand_dims(im, axis=0)
        x = preprocess_input(im)
        preds = model.predict(x)
        result = decode_predictions(preds, top=3)[0]
        resultdict = {}
        if len(result) > 0:
            resultdict[result[0][1]] = float(result[0][2])
        if len(result) > 1:
            resultdict[result[1][1]] = float(result[1][2])
        if len(result) > 2:
            resultdict[result[2][1]] = float(result[2][2])
        else:
            resultdict['null'] = '0'
        return jsonify(resultdict)
```
This method accepts a post request from client, and it will attain a image in the request. Server will call the model to predict the image class and reurn a json text. The top 3 result will be sent. 

Attention，this API router is **/upload**。

run flask，and listen port 5000.

```python
if __name__ == '__main__':
    app.run(host="0.0.0.0",port=5000)
```
The whole code is following, please rename it to `main.py`

```python
from flask import Flask
from flask import request
from flask import jsonify
import numpy as np
import platform
from keras.models import load_model
from keras.preprocessing import image
from keras.applications.resnet50 import preprocess_input, decode_predictions

app = Flask(__name__)

model = None

def init():
    global model
    model = load_model('myresnet.h5')

@app.route('/')
def hello_world():
    a = platform.python_version()
    return a

@app.route('/upload', methods=['POST'])
def upload_file(): 
    if model ==None:
        init()
    if request.method == 'POST':
        f = request.files['the_file']
        imgPath = 'a.jpg'        
        f.save(imgPath)
        im = image.load_img(imgPath, target_size=(224, 224))
        im = image.img_to_array(im)
        im = np.expand_dims(im, axis=0)
        x = preprocess_input(im)
        preds = model.predict(x)
        result = decode_predictions(preds, top=3)[0]
        resultdict = {}
        if len(result) > 0:
            resultdict[result[0][1]] = float(result[0][2])
        if len(result) > 1:
            resultdict[result[1][1]] = float(result[1][2])
        if len(result) > 2:
            resultdict[result[2][1]] = float(result[2][2])
        else:
            resultdict['null'] = '0'
        return jsonify(resultdict)

if __name__ == '__main__':
    app.run(host="0.0.0.0",port=5000)

```

## Write Dockerfile

```dockerfile
FROM gw000/keras:2.1.4-py3-tf-cpu

RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install --upgrade pip

RUN pip3 install \
    flask \
    pillow

RUN mkdir /usr/src/resnet
WORKDIR /workspace
RUN chmod -R a+w /workspace
COPY myresnet.h5 /workspace
COPY main.py /workspace
RUN chmod +x /workspace/main.py
CMD python3 /workspace/main.py

EXPOSE 5000
```

Please pay attention to the code above, Dockerfile create a word directory and  copy `main.py` and `myresnet.h5` to  workspace. When this docker starts, it will run `main.py`.

## Build Docker Image

Please prepare `Dockerfile`,`main.py`,`myresnet.h5` in a separate folder.

```bash
 sudo docker build -t 'flask' .
```
This command will download dependence according to dockerfile.The requirements are enough in this case. In specific cases, requirements should be different.

## Run Docker container

```shell
sudo docker run -it -p 5000:5000 flask
```

Please notice the port and the container's name

If everything is running successfully, you will see

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/run.PNG?raw=true)

Send a post request with an image to our API in Postman.

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/post.PNG?raw=true)

You can see log in console in the meanwhile.

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/output.PNG?raw=true)

If you don't need this resource any longer, you can remove the container directly.



