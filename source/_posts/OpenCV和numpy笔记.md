---
title: OpenCV和numpy笔记
date: 2020-11-22 20:33:53
tags:
- OpenCV
- Numpy
- Python
- 编程记录
---

使用K-means做视频图像分割的作业的时候，遇到一个问题值得记录。

该代码的功能是使用K-means做图像分割并保存图像与视频。

遇到的问题是kmean之后得到的是每一个像素点的分类标签，需要将这些标签可视化成为分割(Segmentation)的结果视频。一开始没有直接找到相关的函数，搜索查询后给的建议都是使用`matlibplot`的`plt.imshow()`函数来实现，但是我不想要每次保存成为图片再转成视频，考虑到opencv-python的底层使用`numpy.array`来表述图像的。一个BGR图像相当于一个3维uint8数组（w,h,c)。要做的就是生成一个标签到像素的映射关系。这里我用的是`matplotlib.cm` color map 功能。

```
x = np.linspace(0.0, 1.0, 10)
rgb = cm.get_cmap(plt.get_cmap('Set1'))(x)[np.newaxis, :, :3][0]
```





```
import numpy as np
from numpy.matlib import repmat
from sklearn.preprocessing import normalize
import matplotlib.pyplot as plt
from matplotlib import cm
from sklearn.cluster import KMeans
import cv2
from PIL import Image

n_cl=5

videoCapture = cv2.VideoCapture('road_video.mov')
fps = videoCapture.get(cv2.CAP_PROP_FPS)
size = (int(videoCapture.get(cv2.CAP_PROP_FRAME_WIDTH)),
        int(videoCapture.get(cv2.CAP_PROP_FRAME_HEIGHT)))
videoWriter = cv2.VideoWriter('output.mp4', cv2.VideoWriter_fourcc(*'mp4v'), (fps/10), size)

def kmeans(data, n_cl, verbose=True):
    n_samples, dim = data.shape
    centers = data[np.random.choice(range(n_samples), size=n_cl)]
    old_labels = np.zeros(shape=n_cl)
    while True:
        distances = np.zeros((n_samples, n_cl))
        for cluster_idx, cluster in enumerate(centers):
            distances[:, cluster_idx] = np.sum(np.square(data - repmat(cluster, n_samples, 1)), axis=1)
        new_labels = np.argmin(distances, axis=1)
        for l in range(0, n_cl):
            centers[l] = np.mean(data[new_labels==l], axis=0)
        if verbose:
            fig, ax = plt.subplots()
            ax.scatter(data[:, 0], data[:, 1], cluster=new_labels, s=40)
            ax.plot(centers[:, 0], centers[:, 1], 'r*', markersize=20)
            plt.waitforbuttonpress()
            plt.close()
        if np.array_equal(new_labels, old_labels):
            break
        old_labels = new_labels
    return new_labels

count = 0
x = np.linspace(0.0, 1.0, 10)
rgb = cm.get_cmap(plt.get_cmap('Set1'))(x)[np.newaxis, :, :3][0]

while True:
    print(count)
    count += 1
#   load the frame
    success, frame = videoCapture.read()
    if not success:
        break
    img = np.float32(frame)
    h,w,c = img.shape
    # print(h,w,c)
#   add coordinates
    row_indexes = np.arange(0, h)
    col_indexes = np.arange(0, w)
    coordinates = np.zeros(shape=(h,w,2))
    coordinates[..., 0] = normalize(repmat(row_indexes, w, 1).T)
    coordinates[..., 1] = normalize(repmat(col_indexes, h, 1))
    data = np.concatenate((img, coordinates), axis=-1)
    data = np.reshape(data, newshape=(w *h, 5))
    labels = kmeans(data, n_cl=n_cl, verbose=False)

    print('after')
    labels = labels.flatten()
    segmented_image = rgb[labels.flatten()]
    # print(segmented_image)
    segmented_image = segmented_image.reshape(img.shape)
    plt.imshow(segmented_image)
    # # plt.imshow(np.reshape(labels, (h, w)), cmap="hsv")
    plt.savefig("lab9/"+str(count-1))
    segmented_image = segmented_image *255
    segmented_image = segmented_image.astype('uint8')
    videoWriter.write(segmented_image)

videoWriter.release()
videoCapture.release()
```

## 参考网页

https://blog.csdn.net/llh_1178/article/details/77833447

https://docs.opencv.org/3.4/d4/dba/classcv_1_1viz_1_1Color.html

https://realpython.com/python-opencv-color-spaces/

https://www.thepythoncode.com/article/kmeans-for-image-segmentation-opencv-python

https://pvss.github.io/Opencv+Python.html

https://stackoverflow.com/questions/9280653/writing-numpy-arrays-using-cv2-videowriter

