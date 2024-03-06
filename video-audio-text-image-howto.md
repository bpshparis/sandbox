# video-audio-text-image-howto

### Check sound and install sound tools

```
systemctl --user status pipewire.service
systemctl --user status pipewire-pulse.service
sudo yum install pavucontrol -y
sudo yum install audacity -y
```

### Install Video

```
sudo dnf install http://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm http://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm -y

sudo dnf install mplayer mencoder -y
sudo yum install ffmpeg -y
```

### Video from image with audio

```
IMAGE="image.png"
VIDEO="video.mp4"
AUDIO="audio.mp3"

ffmpeg -framerate 30 -i ${IMAGE} -c:v libx264 -pix_fmt yuv420p ${VIDEO}

ffmpeg -framerate 1 -pattern_type glob -i '*.png' -i ${AUDIO} -c:v libx264 -r 30 -pix_fmt yuv420p ${VIDEO}
```

### Cut video

```
START="00:01:20"
SOURCE="demo.mp4"
LENGHT="00:08:01"
TARGET="demo-cut.mp4"

ffmpeg -ss ${START} -i ${SOURCE} -c copy -t ${LENGHT} ${TARGET}
```

### Remove audio

```
SOURCE="demo-cut.mp4"
TARGET="demo-cut-vo.mp4"

ffmpeg -i ${SOURCE} -c copy -an ${TARGET}
```

### Merge audio and video

```
VIDEO_SOURCE="demo-cut-vo.mp4"
AUDIO_SOURCE="speech.mp3"
TARGET="merge.mp4"

ffmpeg -i ${VIDEO_SOURCE} -i ${AUDIO_SOURCE} -c:v copy -c:a copy ${TARGET}
```



## OCR

### Install tools

```
sudo yum install ImageMagick -y
sudo yum install tesseract -y
```

### Process images into text

```
for f in $(ls *.png); do convert "$f" "$f.jpg"; done

rm -rf *.png

#Rename to 1.jpg 2.jpg ... n.jpg
i=1 && for f in $(ls *.jpg); do echo "mv" "$f" "$i""-temp.jpg" | sh; ((i+=1)); done

# Force contrast for better result
for jpg in $(ls *.jpg);do echo "Processing" "$jpg..."; echo $jpg | awk -F- '{print "convert -colorspace gray -fill white -resize 200% -sharpen 0x1 " $1"-temp.jpg " $1".jpg"}' | sh; done

rm -rf *-temp.jpg

# OCR step
for jpg in $(ls *.jpg); do echo "Processing" "$jpg..."; echo $jpg | awk -F. '{print "tesseract --dpi 300 " $1 ".jpg " $1}' | sh; done

# Concat all answer in one file
for f in $(ls *.txt); do echo $f | awk -F. '{print "Question N°" $1}' >> questions.txt; echo "" >> questions.txt; cat $f >> questions.txt; echo "" >> questions.txt; echo "Answers:" >> questions.txt; echo "" >> questions.txt; echo 
```



