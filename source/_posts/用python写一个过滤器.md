---
title: 用python写一个过滤器
date: 2019-09-27 17:17:31
tags:
---

计算机视觉课Assigment2的内容.

要求写出一个图像的过滤器出来。

## 要求

**Image Filtering.**Image filtering (or convolution) is a fundamental image processing tool. You will be writing your own function to implement image filtering from scratch. More specifically, you will implement my_imfilter()which imitates the filter2Dfunction in the OpenCV library. As specified in student_code.py, your filtering algorithm must

(1) support grayscale and color images

(2) support arbitrary shaped filters, as long as both dimensions are odd

(e.g. 7x9 filters but not 4x5 filters)

(3) pad the input image with zeros or reflected image content

(4) return a filtered image which is the same resolution as the input image.

使用numpy，PIL。

## 步骤：

1. 导入图像（不属于本次任务）
2. 生成过滤核（不属于本次任务）
3. padding
4. calculate
5. normalization（不属于本次任务，即假设过滤核和图像已经做好了正则化）
6. 截断

---

## 函数定义

图像过滤器的核心是一个function，函数的定义

```python
def my_imfilter(image, filter)
	return filtered_image
```

## 设计思路

输入的图像为image，image可以为黑白或者RGB彩色。（channel=1 or channel=3)

所以输入的image.ndim可能为2或者为3.

第一部分代码需要判断黑白和彩色的情况.

图片过滤器会将原图片的尺寸缩小,因为进行卷积操作是将原图像的一个矩阵框中的点和过滤核相乘后在求和得到新图像的一个点的值。

【公式和图像】

假设原始图像的高和宽为img_h, img_w，过滤核的大小为filter_h, filter_w。这里有一个条件过滤核的长宽都必须为为奇数。（为什么呢？）那么除掉最中心的像素外，过滤核可以被分为四等份。每一部分的长宽(padding)为 pad_h = (filter_h-1)/2和pad_w = (filter_w-1)/2

那么输出的尺寸为img_h - pad_h * 2 和 img_w - pad_w *2，因为图像两边都需要减去pad_h的尺寸。

同时我们还要给过滤器做一个上下左右变换。因为卷积操作和相关操作的过滤核移动方向是相反的，应该是从下往上，从右往左。

## 实例代码：

```python
def my_imfilter(image, filter):
  """
  Apply a filter to an image. Return the filtered image.

  Args
  - image: numpy nd-array of dim (m, n, c)
  - filter: numpy nd-array of dim (k, k)
  Returns
  - filtered_image: numpy nd-array of dim (m, n, c)

  HINTS:
  - You may not use any libraries that do the work for you. Using numpy to work
   with matrices is fine and encouraged. Using opencv or similar to do the
   filtering for you is not allowed.
  - I encourage you to try implementing this naively first, just be aware that
   it may take an absurdly long time to run. You will need to get a function
   that takes a reasonable amount of time to run so that the TAs can verify
   your code works.
  - Remember these are RGB images, accounting for the final image dimension.
  """

  assert filter.shape[0] % 2 == 1
  assert filter.shape[1] % 2 == 1
  # 默认设置图像的通道数，如果只有一个通道，即输入图片是二维的。
  channel = 1
  # 如果输入维度=3，通道数等于第三个维度的元素数量
  if image.ndim == 3:
    channel = image.shape[2]
  # 获取图片的长度和宽度
  image_h, image_w = image.shape[:2]
  # 将过滤核做上下、左右翻转，以确保是卷积操作而不是相关操作
  filter = np.flipud(filter)
  filter = np.fliplr(filter)
  # 获取过滤核的长度和宽度
  filter_h, filter_w = filter.shape
  pad_h = (filter_h - 1) // 2
  pad_w = (filter_w - 1) // 2
  # 先扩充原图像,为了不影响原图像，需要复制一份图像
  image_cp = image.copy()
  image_cp = np.pad(image_cp,[(pad_h,pad_h),(pad_w,pad_w),(0,0)],"constant")
  # 生成过滤后的图片的空容器，尺寸可以是原本的尺寸，在这里为了下面坐标转换方便，扩大了长宽。
  filtered_image = np.zeros(image_cp.shape)
  # 第一层是对于不同的channel做卷积  
  for i in range(channel):
    # 第二层是高度y轴像素遍历
    for j in range(pad_h,image_h+pad_h):
        # 第三层是宽度x轴像素遍历
        for k in range(pad_w,image_w+pad_w):
            # 算法核心，加上上面的翻转式卷积操作，单独来看是相关操作。其实可以通过 step=-1来做
            filtered_image[j,k,i] = np.sum(np.multiply(image_cp[j-pad_h:j+pad_h+1,k-pad_w:k+pad_w+1,i],filter))

  return filtered_image[pad_h:image_h+pad_h, pad_w:image_w+pad_w,:]
```



---

相关链接：

[卷积与互相关的一点探讨](https://zhuanlan.zhihu.com/p/33194385)

