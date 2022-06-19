## 개요 

카메라와 스마트폰으로 촬영한 사진/동영상 파일의 정리와 편집이 끝나면, 백업에 앞서 아래와 같은 후속처리를 하고 있습니다.
- 원본 파일의 파일명은 촬영한 시각으로 변경 (예: IMG_4214.jpg → 20220619 114930.jpg)
- 태블릿/클라우드 저장시 파일용량 축소를 위해 원본 파일과 별도로 사진 라사이징 및 동영상 인코딩

다수의 파일을 다루는 것이기 때문에, 당연히 쉘스크립트를 통해 자동으로 처리합니다. 다른 분들과의 정보공유 차원에서 사진/동영상 파일을 처리하는 전반적인 워크플로우와 함께 쉘스크립트를 소개하고자 합니다. 참고로, [Github Repository](https://github.com/yoonbae81/photo)를 clone하시는 것이 보다 편리할 것입니다.

## 준비 사항

- 스마트폰-컴퓨터간 파일전송을 위한 [FE File Explorer Pro for iOS](https://apps.apple.com/us/app/fe-file-explorer-pro/id499470113) 설치
- 사진 파일의 정보를 추출하고 변경하기 위한 [ExifTool](https://exiftool.org/) 설치
- 사진 파일의 리사이징을 위한 [ImageMagick](https://imagemagick.org/) 설치
- 동영상 파일을 인코딩하기 위한 [FFmpeg](https://ffmpeg.org/) 설치
- Bash 쉘스크립트 실행을 위한 리눅스 또는 윈도우 컴퓨터

## 워크플로우

1. 카메라로 촬영한 사진과 동영상을 스마트폰으로 전송 ([SnapBridge for iOS](https://apps.apple.com/us/app/snapbridge/id1121563450), [Image Sync for iOS](https://apps.apple.com/us/app/image-sync/id959773524) 등)
2. 한 곳에 모인 사진과 동영상을 스마트폰에서 정리
  - 한 장소에서 가장 잘 나온 사진들만 남기고 삭제 ([Photowipe for iOS](https://apps.apple.com/us/app/photowipe/id1137129781))
  - 위치정보가 누락된 파일은 인근에서 촬영한 사진의 [위치정보를 확인](https://support.apple.com/ko-kr/guide/iphone/iph390138909/ios)해 누락된 파일에 붙여넣기
  - 사진파일 보정 및 동영상 편집(필요시)
3. 정리된 사진들은 스마트폰에서 컴퓨터로 전송
4. 컴퓨터에서 쉘스크립트 실행
  - 촬영한 시각으로 원본 파일의 파일명 변경
  - 태블릿/클라우드 저장을 위한 파일용량 축소
5. 처리가 끝난 원본 파일은 외장하드에 백업, 리사이징/인코딩된 파일은 태블릿/클라우드 등에 업로드

## 파일명 변경 쉘스크립트

스마트폰에서 전송한 파일의 [EXIF](https://exiftool.org/TagNames/EXIF.html)에서 촬영시각을 추출해 [`20220619 114930.jpg`](https://exiftool.org/faq.html#Q5)와 같은 형태로 파일명을 바꾸어 저장합니다. 사진의 촬영시각은 DateTimeOriginal에서, 동영상은 CreationDate에서 추출합니다. 

참고로, ExifTool을 활용해 파일에 담긴 모든 시간정보를 확인하려면 `exiftool -time:all -a -G -s FILE`를 입력하면 됩니다.

```
#!/bin/bash

if [ ! $# -eq 2 ]; then
    echo "Usage: $0 source target"
    exit 1
fi

if [ ! -d $1 ]; then
    echo "Not found: $1"
    exit 1
fi

if [ -f "$1/_RENAMED" ]; then
    echo "It seems the previous work hasn't confirmed after its completion as _RENAMED exists in source directory"
    exit 1
fi

if [ ! -d $2 ]; then
    echo "Making target directory: $2"
    mkdir -p $2
fi

echo "==== Renaming jpg files with DateTimeOriginal"
exiftool '-FileName<DateTimeOriginal' -dateFormat '%Y%m%d %H%M%S%%3nc.%%e' -IPTCDigest=new -fixBase -extractEmbedded -ext jpg -out $2 $1

echo "==== Renaming mov/avi/mp4 files with CreationDate"
exiftool '-FileName<CreationDate' -dateFormat '%Y%m%d %H%M%S%%3nc.%%e' -IPTCDigest=new -fixBase -preserve -extractEmbedded -ext mov -ext avi -ext mp4 -out "$2" "$1"

touch "$1/_RENAMED"
exit 0
```

## 사진 리사이징 및 동영상 인코딩 쉘스크립트

그동안 촬영한 모든 사진/동영상을 모든 기기들에서 감상하려면, 모든 기기를 고용량으로 구입하거나 매달 고용량의 클라우드 서비스를 구독해야 합니다. 가족 수와 보유 기기의 수를 감안하면 제법 큰 비용이 소요되기 때문에, 원본은 별도로 보관하되 감상용 사진/동영상은 과감히 리사이징/인코딩해 용량을 줄이기로 했습니다. 

참고로, 사진 리사이징은 [여러 종류의 아이폰/아이패드 해상도](https://www.ios-resolution.com/) 가운데, 아이패드 미니 5세대의 좁은 폭인 1536 픽셀로 정했습니다. 동영상 인코딩은 몇차례 테스트를 거쳐 [H.265/HEVC](https://en.wikipedia.org/wiki/High_Efficiency_Video_Coding) 코덱의 중간 수준 압축률로 정했습니다. 그 결과, 원본 폴더의 용량은 104 GB, 리사이즈 폴더의 용량은 24 GB로, 어느 기기에 저장해도 부담없는 수준이 되었습니다.

```
#!/bin/bash

if [ ! $# -eq 2 ]; then
    echo "Usage: $0 source target"
    exit 1
fi

if [ ! -d $1 ]; then
    echo "Not found: $1"
    exit 1
fi

if [ -f "$1/_RESIZED" ]; then
    echo "It seems the previous work hasn't confirmed after its completion as _RESIZED exists in source directory"
    exit 1
fi

if [ ! -d $2 ]; then
    echo "Making target directory: $2"
    mkdir -p $2
fi

shopt -s nullglob
shopt -s nocaseglob

echo "==== Resizing jpg files"
for path in $1/*.jpg; do
    f="${path##*/}"
    magick "$path" -resize "1536x1536^>" -verbose "$2/$f"
done

echo "==== Encoding mov/avi/mp4 files"
for path in $1/*.{mov,avi,mp4}; do
    f="${path##*/}"
    ffmpeg -y -i "$path" -map_metadata 0 -movflags use_metadata_tags -c:a aac -b:a 128k -c:v libx265 -crf 25 -tag:v hvc1 -hide_banner -loglevel error "$2/$f"
done

echo "==== Refreshing EXIF tags including GPS position after HEVC encoding"
exiftool -if '$GPSPosition' -if 'not $UserData:GPSCoordinates' '-UserData:GPSCoordinates<GPSPosition' '-UserData:Make<Make' '-UserData:Model<Model' -preserve -extractEmbedded -overwrite_original -ext mov -ext avi -ext mp4 "$2"

touch "$1/_RESIZED"
exit 0
```

만약, 윈도우에서 한 디렉토리에 들어 있는 여러 파일에 대해 반복적으로 명령어를 처리하기 위해서는 아래의 `FOR`를 활용하면 됩니다. 만약 배치파일에 넣어 동작시킬 경우 모든 %를 %%로 변경해야 합니다.
```
FOR /F "tokens=*" %G IN ('dir /b *.mov *.mp4 *.avi') DO ffmpeg -i "%G" -map_metadata 0 -movflags use_metadata_tags -c:a aac -b:a 128k -c:v libx265 -crf 25 -tag:v hvc1 "encoded\%~nG.mp4"
```

## 자동화

상기 쉘스크립트를 매번 실행하는 것도 상당히 귀찮은 일입니다. 따라서, [crontab](https://wiki.archlinux.org/title/cron)에 아래와 같이 매시간마다 NAS의 폴더(Inbox)에 새로운 파일이 저장되면 자동으로 처리하도록 했습니다. 여기까지 읽으셨다면 비로소 전체적인 워크플로우와 구체적인 쉘스크립트 실행방식을 모두 이해하실 수 있을 것입니다.

```
30 7-23 * * * ~/photo/rename /mnt/2tb/Pictures/Inbox /mnt/2tb/Pictures/`date +\%Y-\%m-\%d`
35 7-23 * * * ~/photo/resize /mnt/2tb/Pictures/`date +\%Y-\%m-\%d` /mnt/2tb/Resized/`date +\%Y-\%m-\%d`
```

## 참고사항

### ExifTool

EXIF에 별다른 문제는 없는지 Validate하는 명령어입니다.
```
exiftool -validate -warning -a FILE
```

Validate 결과 문제가 많다면, EXIF를 Rebuild하는 명령어입니다.
```
exiftool -all= -tagsfromfile @ -all:all --padding -unsafe -m -F FILE
```

그동안 사진/동영상 파일의 EXIF 태그에 불필요한 정보들이 많이 누적되어 있을 경우, 모두 삭제해주는 명령어입니다.
```
exiftool -a -s -F -P -r -rating= -label= -description= -keywords= -subject= -HierarchicalSubject= -RegionName= -CodedCharacterSet=utf8 -IPTCDigest=new -overwrite_original -ext jpg -ext avi -ext mov -ext mp4 .
```

### H.265/HEVC 인코딩

[H.265/HEVC](https://en.wikipedia.org/wiki/High_Efficiency_Video_Coding) 코덱으로 인코딩시 카메라 및 촬영위치가 아이폰에서 제대로 표시되지 않는 문제가 있습니다. 따라서, Make, Model, GPSCoordinates 등의 정보를 인코딩 이후 다시 한 번 UserData 태그에 적어주는 절차가 필요합니다. 상기 쉘스크립트에는 이미 반영되어 있습니다. 만약 윈도우에서 실행시 -if 뒤는 '가 아닌 "로 대체해야 합니다.
```
exiftool -if '$GPSPosition' -if 'not $UserData:GPSCoordinates' '-UserData:GPSCoordinates<GPSPosition' '-UserData:Make<Make' '-UserData:Model<Model' -preserve -extractEmbedded -overwrite_original -ext mov -ext avi -ext mp4 .
```

### [Geotagging](https://en.wikipedia.org/wiki/Geotagging)

모든 사진/동영상 파일 가운데 단 하나의 파일도 빠짐없이 위치정보를 입력하는 것은 과감한 결단이 요구되는 지난한 과정이 될 것입니다. 약 보름에 걸쳐 이를 완성하고 Photos 앱에서 지도에 표시된 사진들을 감상하고 있자니 뿌듯하기 이를 데 없습니다. Geotagging 노하우는 처리할 작업량의 규모에 따라 나누어 설명할 수 있을것 같습니다.

#### 위치정보가 누락된 파일의 수가 많지 않을때

파일에 GPS 태그를 확인하는 명령어입니다.
```
exiftool -G1 -a -n -"GPS*" FILE
```

같은 장소에서 찍은 다른 사진의 GPS 좌표를 다른 파일에 입력하는 명령어인데, 사진의 양이 많지 않다면, 아이폰에서 위치정보가 지정된 사진의 지도를 탭&홀드하여 위치정보를 복사하고, 위치정보가 누락된 사진의 지도에 붙여넣기 하는 방법이 더 편리합니다.
```
exiftool -overwrite_original -wm cg -TagsFromFile source.jpg -"UserData:GPSCoordinates<GPSPosition" FILE
```

만약 인근 지역에서 촬영한 사진은 없지만, [지도를 통해 좌표를 직접 얻어낼 수 있는 경우](https://support.google.com/maps/answer/18539?hl=en&co=GENIE.Platform%3DDesktop), 아래와 같은 명령어를 활용하면 됩니다. 
```
exiftool -overwrite_original -Keys:GPSCoordinates="35.33702233389209, 129.3100879767233" *.mp4
```

#### 몇일간의 여행에서 찍은 사진 가운데 일부 파일의 위치정보가 누락되어 있을때

스마트폰으로 촬영한 사진들은 언제나 위치정보를 저장하고 있으나, 카메라로 촬영한 사진들은 스마트폰과 싱크할때 위치정보를 제대로 전달받지 못해 위치정보가 누락되는 경우가 많습니다. 이 경우, 위치정보를 포함하는 사진들로부터 시간대별 위치정보를 뽑아내 [GPX](https://en.wikipedia.org/wiki/GPS_Exchange_Format) 파일에 저장하고, 이 파일을 토대로 위치정보가 누락된 파일에 위치정보를 계산해 넣어주는 [Inverse Geotagging](https://exiftool.org/geotag.html#Inverse) 방식이 있습니다.

```
exiftool -r -m -if "$GPSDateTime" -fileOrder GPSDateTime -p gpx.fmt . > track.gpx
exiftool -geotag=track.gpx -api GeoMaxIntSecs=5184000 -overwrite_original -if "not $GPSPosition" -ext jpg .
```

#### 수개월/수년간의 사진들에 위치정보가 누락되어 있을 때

윈도우용 [GeoSetter](https://geosetter.de/en/main-en/)을 추천합니다. 다수의 파일에 한번에 좌표를 입력할 수 있게 도와줍니다. 다만, 2019년 이후 업데이트가 안되어 지도의 검색기능이 제대로 동작하지 않기 때문에, [구글 지도의 좌표](https://support.google.com/maps/answer/18539?hl=en&co=GENIE.Platform%3DDesktop)를 가져와서 GeoSetter의 즐겨찾기에 저장해 놓고 활용하면 좋습니다.
 
![](geosetter.png#center)

참고로, 아래는 미리 설정해 놓으면 편리한 환경설정 변수들입니다.
```
- File Operations
  - Overwrite Original File when Saving Changes: Enabled
  - Preserve File Date and Time when Saving Changes: Enabled
- Data Preferences
  - Save Time Zone to Exif Data: Enabled
  - Set Taken Date to all Exif Dates: Enabled
  - Add Time Zone Automatically to Taken Date when Assigning Map Position: Enabled
```

## References

- [ExifTool Application Documentation](https://exiftool.org/exiftool_pod.html)
- [ExifTool Tags](https://exiftool.org/TagNames/EXIF.html)
- [ExifTool GPS Tags](https://exiftool.org/TagNames/GPS.html)
- [ExifTool Geotagging](https://exiftool.org/geotag.html)
- [ExifTool Shortcuts Tags](https://exiftool.org/TagNames/Shortcuts.html)
- [ExifTool Copying examples](https://exiftool.org/exiftool_pod.html#COPYING-EXAMPLES)
- [Diff Checker](https://www.diffchecker.com/diff)
- [Using FFmpeg to Create HEVC Videos That Work on Apple Devices (aaron.cc)](https://aaron.cc/ffmpeg-hevc-apple-devices/)
