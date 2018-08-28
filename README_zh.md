# Deploy pre-trained model with flask on Docker

> 本文介绍了使用flask将预训练的模型发布到Docker上成为Web API的具体步骤

## 开始之前

请确保您的计算机满足以下条件后再开始执行之后的操作。

- 已在设备上部署Docker并熟悉Docker命令
- 熟悉Python
- 已有预训练好的模型文件（本文使用的是ResNet)
- Azure Data Science Virtual Machine (可选)

## 编写flask服务端代码

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

此方法接受客户端的POST请求，并在POST请求中获得图片，使用模型对图片类别进行预测后返回JSON序列化后，预测结果概率最高的三项。

请注意，此API路由为**/upload**。

启动flask，并监听5000端口。

```python
if __name__ == '__main__':
    app.run(host="0.0.0.0",port=5000)
```

以下为完整代码，将文件命名为`main.py`

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

## 编写Dockerfile

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

请注意Dockerfile中创建了相应工作目录，并将`main.py`和模型文件`myresnet.h5`复制到了相应目录下。启动此Docker镜像时运行`main.py`即可。

## 构建Docker Image

请在单独文件夹中准备好需要的三个文件`Dockerfile`,`main.py`,`myresnet.h5`。

```bash
 sudo docker build -t 'flask' .
```

命令将根据Dockerfile中的需求下载对应依赖和python包，对于此应用，使用上面的Dockerfile依赖已经足够，对于具体应用请根据需求变更。

## 启动Docker容器

```shell
sudo docker run -it -p 5000:5000 flask
```

请注意匹配端口号和Docker镜像名称。

成功启动后将看到如下输出：

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/run.PNG?raw=true)

使用Postman向API发送带有图片的POST请求等待结果:

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/post.PNG?raw=true)

同时能在终端中查看即时命令输出:

![](https://github.com/Iamnvincible/Deploy_Model_on_Docker/blob/master/imgs/output.PNG?raw=true)

如果不再需要此资源可直接删除对应Docker容器。



