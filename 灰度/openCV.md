## [用opencv的traincascade.exe训练行人的HAAR、LBP和HOG特征的xml](https://www.opencv.org.cn/opencvdoc/2.3.2/html/doc/user_guide/ug_traincascade.html)

**imagemagick**灰度化

 convert *.png -colorspace GRAY -contrast-stretch 2%x35% gray%02d.png

```
convert in.jpg -monochrome out.jpg
 convert *.png -monochrome monochrome%02d.png
 convert *.png -threshold 55% monochrome%02d.png
```

```
D:\Tesseract\image\opencv
					neg
					pos
					xml
```

dir /s/b > pos.txt

dir /s/b > neg.txt

替换pos.txt中的.png --> .png 1 0 0 24 24

将pos.txt和neg.txt复制到工作目录下

D:\Tesseract\image\opencv>opencv_createsamples.exe -info pos.txt -vec pos.vec -bg neg.txt -num 113 -w 24 -h 24

opencv_traincascade.exe -data xml -vec pos.vec -bg neg.txt -numPos 100 -numNeg 60 -numstages 20 -featureType LBP -w 24 -h 24

opencv_traincascade.exe -data xml -vec pos.vec -bg neg.txt -numPos 150 -numNeg 330 -numstages 20 -featureType LBP -w 24 -h 24



### [Python用*dilb*](http://www.baidu.com/link?url=8wk4V5u1FcVyMzQ9d9-c3t1JFC3TEL0DhUmzc0APUjM9KNpkMOgI51Xlxvov7MIF8BOopfVExj8tYMeEeDCijq)







