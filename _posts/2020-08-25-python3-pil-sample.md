## python3 pil库使用
    好久没有写python了，为啥突然写一篇了，因为我在阿里云申请了一个域名，但是呢需要实名认证，实名认证又需要把身份证图片上传下，
发现上传的图片是合并到一起的，那很坑，为啥就不能上传两张，正反面嘛，所以没办法用python搞下吧，应该就十几行代码。

### PIL库简介
    
PIL(Python Imaging Library)是Python一个强大方便的图像处理库，名气也比较大。不过只支持到Python 2.7。
 
PIL官方网站：http://www.pythonware.com/products/pil/
 
Pillow是PIL的一个派生分支，但如今已经发展成为比PIL本身更具活力的图像处理库。目前最新版本是3.0.0。
 
Pillow的Github主页：https://github.com/python-pillow/Pillow
Pillow的文档(对应版本v3.0.0)：https://pillow.readthedocs.org/en/latest/handbook/index.html

安装,用国内pip源，连不上默认的。。：
```bash
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pillow
```

### 代码，简单的图片合并，及模式转换
```python
from PIL import Image


# need the same type and size
def rebuild_images_to_one_pic(*images):
    if len(images) == 0:
        return
    wigth, height = images[0].size
    new_images = []
    # resize the images to the same size
    for i in images:
        newimage = i.resize((wigth, height), Image.BILINEAR)
        new_images.append(newimage)

    new_one_image = Image.new(new_images[0].mode, (wigth, height * len(new_images)))

    for i, im in enumerate(new_images):
        new_one_image.paste(im, box=(0, i * height))
    return new_one_image


# 1 (1-bit pixels, black and white, stored with one pixel per byte)
# L (8-bit pixels, black and white)
# P (8-bit pixels, mapped to any other mode using a color palette)
# RGB (3x8-bit pixels, true color)
# RGBA (4x8-bit pixels, true color with transparency mask)
# CMYK (4x8-bit pixels, color separation)
# YCbCr (3x8-bit pixels, color video format)
# LAB (3x8-bit pixels, the L*a*b color space)
# HSV (3x8-bit pixels, Hue, Saturation, Value color space)
# I (32-bit signed integer pixels)
# F (32-bit floating point pixels)
# supported modes
# 转换为黑白的。
def change_mode(image, mode):
    new = image.convert(mode)
    new.save("/Users/zhouxiaoming/test.jpg")


if __name__ == "__main__":
    image1 = "/Users/zhouxiaoming/WechatIMG2.jpeg"
    image2 = "/Users/zhouxiaoming/WechatIMG1.jpeg"
    new_image = rebuild_images_to_one_pic(Image.open(image1), Image.open(image2))
    new_image.save("/Users/zhouxiaoming/new_id.jpg")
    change_mode(new_image, "L")

```

总共花了半个小时以上了。。。
    
    
