---
section: post
date: "2018-02-05"
title: "Object Detection using Single Shot Detection Algorithm"
description: "Using tensorflow to build a chatbot demonstrating the power of Deep Natural Language Processing"
slug: "object-detection-using-single-shot-detection-algorithm"
tags:
 - computer_vision
 - ssd
 - tensorflow
 - project
 - deep_learning
 - tutorial 
---
<!-- # How a software developer re-focused his life to learn artificial intelligence -->

![Computer Vision](/images/articles/2018/computer_vision/object-detection-recognition-and-tracking-intro.jpg "Computer Vision Intro")
<center>image borrowed from <a href="https://software.intel.com/en-us/articles/a-closer-look-at-object-detection-recognition-and-tracking">Intel Developer Site</a></center>

# Addon Prelude
``Read the article from intel developers zone.``<br/>
https://software.intel.com/en-us/articles/a-closer-look-at-object-detection-recognition-and-tracking

---

# Prelude

I have taken the explanation from <a href="https://towardsdatascience.com/understanding-ssd-multibox-real-time-object-detection-in-deep-learning-495ef744fab">towardsdatascience.com - Understanding SSD MultiBox — Real-Time Object Detection In Deep Learning</a> and further simplified it for much better understanding. Especially for the slow ones like me :P

## Architecture of SSD

![som_architecture.png](/images/articles/2018/computer_vision/som_architecture.png "som_architecture.png")
<center>image borrowed from <a href="https://arxiv.org/pdf/1512.02325.pdf">https://arxiv.org/pdf/1512.02325.pdf</a></center>

### mean Average Precision (mAP)
Lets try to understand what is Average Precision:<br/>e.g. Let’s say, we recommended 7 products to a customer and the 1st, 4th, 5th, 6th product recommended was correct. So now the result would look like - 1, 0, 0, 1, 1, 1, 0. <br/>

In this case,

- The precision at 1 will be: 1/1 = 1
- The precision at 2 will be: 0/2 = 0
- The precision at 3 will be: 0/3 = 0
- The precision at 4 will be: 2/4 = 0.5
- The precision at 5 will be: 3/5 = 0.6
- The precision at 6 will be: 4/6 = 0.66
- The precision at 7 will be: 0/7 = 0

Average Precision will be: 1 + 0 + 0 + 0.5 + 0.6 + 0.66 + 0 /4 = 0.69 — Please note that here we always sum over the correct images, hence we are ``dividing by 4 and not 7``.
MAP is just an extension, where the mean is taken across all AP scores.

[thanks to Pallavi Sahoo](https://medium.com/@pds.bangalore/mean-average-precision-abd77d0b9a7e)

The paper about [SSD: Single Shot MultiBox Detector](https://arxiv.org/abs/1512.02325) (by C. Szegedy et al.) was released at the end of November 2016 and reached new records in terms of performance and precision for object detection tasks, scoring over 74% mAP (mean Average Precision) at 59 frames per second on standard datasets such as [PascalVOC](http://host.robots.ox.ac.uk/pascal/VOC/) and [COCO](http://cocodataset.org/#home). To better understand SSD, let’s start by explaining where the name of this architecture comes from:

# Lets Start!

---

# Importing Libraries

```py
import torch
from torch.autograd import Variable
import cv2
from data import BaseTransform, VOC_CLASSES as labelmap
from ssd import build_ssd
import imageio
```

- ``Torch``: Library that cointain ``PyTorch`` - it contains the dynamic graphs for efficient calculation of the gradient of composition functions in backpropagation(computing weights).
- ``torch.autograd``: module responsible for gradient decent
- ``torch.autograd import Variable``: used to convert Tensors into Torch Variables that contains both the tensor and the gradient.
- ``cv2``: to draw rectangles on images not the detection
- ``data``: is just a folder containing the classes BaseTransform, VOC_CLASSES (pretrained model using CUDA)
- ``BaseTransform``: is a class for image transformationns making the input images compatible with neural network
- ``VOC_CLASSES``: for encoding of classes eg: planes as 1, dogs as 2 so we can work with numbers and not texts
- ``ssd``: library of the single shot multibox detector
- ``build_ssd``: is the constructor to build the architecture of single shot multibo xarchitecture.
- ``imageio``: library to process the images of the video (an alternative to [PIL](https://pillow.readthedocs.io/en/latest/))

# Building a Function for Object Detection

Now we are going to do a frame by frame detection i.e we user ``imageio`` library to extract all the frames calculating ``fps`` (frames per second) - then do the object detection and stitch back all frames to a video.

We will create a function to do all there operation called ``detect`` which will return the ``frame`` containing the rectangle on the detected image and its label,

```py
def detect(frame, net, transform):
``` 
- ``frame``: image on which the detect function will be applied
- ``net``: this will be the single shot multibox detector nueral network
- ``transform``: transform the input images so that they are compatible with the network

Now lets work on the first input the ``frame``.

We need to get the height and weight of the image. We need to take this from the frame and it has as attribute ``.shape`` which returns a vector of three elements [height, weight, number_of_channels(1 for black and white & 3 for color)]

```py
height, width = frame.shape[:2] #range 0 to 2 except 2
```

# Image Transformations

There are 4 transformations that we need to apply on to the image(frame)

i.e original image(frame) => Torch varible compatible with Nueral Network.

1. Is to apply the ``transform`` transformation to make sure the image has the right dimensions and color value.
2. Convert this transformed frame from ``numpy array`` to ``torch_tensor``
3. Add a fake dimention to ``torch_tensor`` for batch
4. Convert it to a torch variable(both tensor and gradient)

1.
```py
frame_transformed = transform(frame)[0] # returns 2 elements. we need only the transformed frame of index [0]
```
2.
```py
x = torch.from_numpy(frame_transformed).permute(2,0,1) # the pre-trained SSD model was done in GRB format not in RGB. Hence the conversion.
```
3.The neural network cannot accept single input vector or image it only accepts in batches.
So now we need to create a structure with the first dimension as the batch and other dimension as the input.
```py
(x.unsqueeze(0) # 0 is the index of the batch and batch should be the first index in the tensor.
```
4.Convert this batch of torch tensor of inputs to a torch variable.
<br/> A torch variable is a highly advanced variable that containes both a tensor and the gradient.<br/> This torch variable will become an element of the dynamic graph which will conpute the gradients very efficiently of any backpropagation 
```py

```

Its time to feed our ``torch variable`` to the SSD Neural Network

```py
y = net(x)
```

Now we have the output ``y``. This y directly does not contain what we are interested in i.e the result of the detection wheather we have a dog or a human in frame. So to get the specific information from y we need to use the ``data`` attribute from y

```py
detections = y.data
```
Now we need to create a new tensor which will have dimention as [width, height, width, height].<br/> This is because the position of the detected object inside the image has to be normalized between 0 & 1 and to do this normalization we will need this scaled tensor with these 4 dimensions.

The first 2 width & height corresponds to the scalar values of the upper left corner

```py
scale = torch.Tensor([width, height, width, height])
```

The detection tensor contains 4 elements
batch: we created the fake dimension with unsqueeze
number of classes: the objects that can be detected like dog, place, boat, car
number of occurance of the class: count of the previous classes. like 2 dogs in a frame.
tuple: 5 element tuple - score, x0, y0, x1, y1 - for each occurance we get a score and its cordinates upper left corner and lower right corner. score(threshold) > 0.6 to be found.

```py
for i in range(detections.size(1)): #detection(size(i)) is the number of classes
    j = 0 # occurances of class i
    while detections[0, i, j, 0] >= 0.6: # for score >= 0.6 [batch,class,occurance, score]
        points = (detections[0, i, j, 1:] * scale).numpy() #here we are not interested in score but the cordinates hence 1: - scale(normalize) and convert to numpy array for openCV
        #draw rectangle - frame color red - thickness of 2
        cv2.rectangle(frame, (int(points[0]), int(points[1])), (int(points[2]), int(points[3])), (255, 0, 0), 2)
        #print the label - labelmap (to get the class text)is the dictionary from VOC_CLASSES we imported - i-1 is for phthon index 0 - then font - size - color - continues text not dots.
        cv2.putText(frame, labelmap[i-1], (int(points[0]), int(points[1])), cv2.FONT_HERSHEY_SIMPLEX, 2, (255, 255, 255), 2, cv2.LINE_AA)
        j += 1 # increment j
return frame # return of the for loop.
```

# Creating the SSD Neural Network

```py
net = build_ssd('test') # test phase as we are using a pre-trained model from .pth file next.
```

# load the weights from already pretrained NN
ssd300_mAP_77.43_v2.pth is pre-trained of about 30-40 objects.

```py
#to open a tensor containing weights
net.load_state_dict(torch.load('ssd300_mAP_77.43_v2.pth', map_location = lambda storage, loc: storage))
```

#Transformation

```py
#Making the frame is compatible with the neural network.
transform = BaseTransform(net.size, (104/256.0, 117/256.0, 123/126.0)) #net.size is  the target size of the images, tupple of 3 arguments - taken from the pretrained network (under certain convention for color values.)
```

# Doing object detection in video

```py
reader = imageio.get_reader('funny_dog.mp4')
fps = reader.get_meta_data()['fps']
writer = imageio.get_writer('output.mp4', fps = fps)
for i, frame in enumerate(reader):
    processed_frame = detect(frame, net.eval(), transform)
    writer.append_data(processed_frame)
    print(i)
writer.close()
```

# Final Output

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/n7z0Y3BHiVY?rel=0&amp;controls=0&amp;showinfo=0" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe><center>

# Source Code
<center> <a href="https://goo.gl/rXCEPB"><img src="/images/GitHub.jpg"></a> </center>




