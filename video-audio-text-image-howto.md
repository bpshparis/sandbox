# Video-audio-text-image-howto

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

### Capture screen with sound

```
sudo yum  install xwininfo -y
```
:bulb: grab last line output (e.g. -geometry 1920x1022+0+0) to set -video_size and value after + for -i parameter below

```
VIDEO="video.mp4"

ffmpeg -video_size 1900x1080 -framerate 25 -f x11grab -i :0.0+0,0 -f pulse -ac 2 -i default ${VIDEO} -y
```

### Capture screen with sound

```
#!/bin/sh
xwininfo | {
    while IFS=: read -r k v; do
        case "$k" in
        *"Absolute upper-left X"*) x=$v;;
        *"Absolute upper-left Y"*) y=$v;;
        *"Border width"*) bw=$v ;;
        *"Width"*) w=$v;;
        *"Height"*) h=$v;;
        esac
    done
    for i in 3 2 1; do echo "$i"; sleep 1; done
    ffmpeg -y -f x11grab -framerate 30 \
           -video_size "$((w))x$((h))" \
           -i "+$((x+bw)),$((y+bw))" screenrecord.mp4
}

```

### Install youtube-dl

```
git clone https://github.com/ytdl-org/youtube-dl
cd youtube-dl
make
sudo cp -v youtube-dl /usr/local/bin/
```

💡 Don't pay attention to **make: pandoc: No such file or directory**, **youtube-dl** binary should be generated.


### Download youtube videos

```
VIDEO="https://www.youtube.com/watch?v=ywobzuCN158"
BROWSER="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
youtube-dl --user-agent "${BROWSER}" ${VIDEO}
```

### Install yt-dlp (france.tv, arte.tv)

```
cd ~/Videos
wget -c https://github.com/yt-dlp/yt-dlp/releases/download/2024.10.07/yt-dlp_linux
chmod +x yt-dlp_linux

URL="https://www.arte.tv/fr/videos/116856-000-A/angkor-et-le-tresor-oublie-des-khmers"
FFMPEG_PATH="/usr/bin/ffmpeg"

./yt-dlp_linux -i --no-abort-on-error -q --no-warnings --no-sponsorblock --cookies-from-browser firefox --no-check-certificates --video-multistreams --audio-multistreams --prefer-free-formats --merge-output-format mkv --ffmpeg-location ${FFMPEG_PATH} --allow-dynamic-mpd --hls-use-mpegts --sub-format best --sub-langs all --embed-subs ${URL}
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

> 💡 [calculatrice-temps](https://www.ma-calculatrice.fr/calculatrice-temps)

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



### Shrink video size

```
VIDEO_SOURCE="queen-love-of-my-life.webm"
VIDEO_TARGET="queen-love-of-my-life.mp4"

ffmpeg -i ${VIDEO_SOURCE} -c:v libx265 -crf 28  -c:a copy ${VIDEO_TARGET}
```

### Shrink video size - Alternate

```
SOURCE="video.mp4"
TARGET="Video.mp4"
WIDTH="640"

ffmpeg -i "${SOURCE}" -vf "scale=${WIDTH}:trunc(ow/a/2)*2" "TARGET" -y

NEW_RES=$(ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "${TARGET}")

SHRINKED=$(echo ${TARGET} | sed 's:\.:-'${NEW_RES}'.:')

mv ${TARGET} ${SHRINKED}
```



### Extract video and encode to mp3

```
VIDEO_SOURCE="queen-love-of-my-life.webm"
AUDIO_TARGET="queen-love-of-my-life.mp3"

ffmpeg -i ${VIDEO_SOURCE} -vn -ab 128k -ar 44100 -y ${AUDIO_TARGET}
```

## IMAGE

:bulb: First, you need to convert the image format from .jpg to .png format, because JPEG does not support transparency.

### Install tools

```
sudo yum install ImageMagick -y
```

### Sharpen
```
IMG="signature"

magick $IMG.jpg -colorspace gray -fill white -resize 400% -sharpen 0x5 ${IMG}-sharpen.png
```

### Transparency
```
IMG="signature"

magick ${IMG}-sharpen.png -fuzz 70% -transparent white ${IMG}-transparent.png
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

## QRCODE

### Install tools

```
sudo yum install qrencode eog zbar ImageMagick -y
```

### Generate qrcode

#### Downloadable link

```
MYEXTIP=$(curl -4  ifconfig.me)
URL="http://${MYEXTIP}:5102/sharing/xej5fQx6t"
IMG="dexter-new-blood.png"

qrencode -o ${IMG} ${URL}
```

#### Downloadable link

```
MYEXTIP=$(curl -4  ifconfig.me)
URL="http://${MYEXTIP}:5102/sharing/xej5fQx6t"
IMG="dexter-new-blood.png"

qrencode -o ${IMG} ${URL}
```

#### WIFI access

> :bulb: WIFI:S:{SSID name of your network};T:{security type - WPA or WEP};P:{the network password};;

```
IMG="MyWifiAccess"
SSID="Livebox-AA"
CODE="ABC123"

echo -n "WIFI:S:${SSID};T:WPA;P:${CODE};;" | qrencode -t ansiutf8

qrencode -o ${IMG}-temp.png "WIFI:S:${SSID};T:WPA;P:${CODE};;"

magick ${IMG}-temp.png -colorspace gray -fill white -resize 200% -sharpen 0x1 ${IMG}.png
```

### Display qrcode

```
IMG="dexter-new-blood.png"

eog ${IMG}
```

### Decode qrcode

```
IMG="dexter-new-blood.png"

zbarimg ${IMG}
```

