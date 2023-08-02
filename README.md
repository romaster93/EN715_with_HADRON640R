# Carrier board setting

We are using a some carrier board for Jetson series.

Because we need more socket or function.

So,now i need to use a EN715 carrier board of Avermedia for LIDAR SLAM and using hadron640r camera.

Lets start <br><br>


### 1.  **설치사전준비** <br><br>
먼저 내장 메모리에 설치를 진행하게되면 거의 대부분 사용이 불가능할거라도 생각한다. <br>
따라서 내장메모리에 설치가 아닌 sd card 또는 외장 ssd에 설치를 진행한다. <br>
(사실 hadron 카메라가 jetson nano 밖에 아직 지원을 안해줘서 억지로 nano를 사용하는거지 카메라만 아니면 바로 orin nx로 넘어가고싶다....;)<br>
<br>
우선 BSP 파일이 필요한데 일반 사용자라면 에버미디어 홈페이지에서 원하는 버전으로 다운받으면 된다. <br>
링크 : https://www.avermedia.com/professional/download/en715#parentHorizontalTab3 <br>
<br>
나는 HADRON 640R 제품을 지금 꼭 사용해야해서 해당 제품의 드라이버를 지원하는 버전인 Jetpack4.3 버전을 사용한다.<br>
(내가 사용한 버전은 github에 업로드 할 예정)<br>
그리고 HADRON 640R 카메라를 사용하기 위해서 Image 파일과 dtb 파일을 바꿔줘야하는데 이것도 gtihub에 업로드 해둘 예정이다.<br><br>
<span style=color:orange>[HADRON 640R 사용자라면 아래 과정을 추가로 실시]</span><br><br>
모든 파일이 준비되었다면 BSP 파일에서 <br>
"Image" 파일 &rarr; /Linux_for_Tegra/kernel <span style=color:grey>(이때 image파일이 두개가있을텐데 (이름이 다른걸로) 둘중에 뒤에 이름이 더 긴걸로 선택하면 된다)</span><br>
"dtb" 파일 &rarr; /Linux_for_Tegra/kernal/dtb <br><br>
전부 덮어씌우고 나면 설치 진행을 하면된다.<br><br>




### 2.  **SD카드에 플래쉬 준비** <br><br>
SD카드를 일반 PC에 연결하고 SD카드 초기화를 진행 후 ext4 파일 시스템을 생성해서 부팅 디스크로 만들어준다. <br>
우선 SD카드를 연결하면 /dev/sd(x)로 나오게 되는데 그걸 기억한다.<br>

    export sdcard=/dev/sd(x)
    sudo gdisk $sdcard
        "o"                    #clear all current partition data
        "n"                    #create new partition
        "1"                    #partition number /dev/sdx1, 만약 이미 있다고 하면 해당 파티션 삭제 후 다시 진행
        "40M"                  #Press enter or "+32G" last sectors, 40M 입력해서 사용가능
        "Linux filesystem"     #using default type, 기본값 사용가능
        "c"                    #partition's name "RootFS", 이건 원하는대로 입력가능
        "w"                    #write to disk and exit
    
    sudo mkfs.ext4 $sdcard\1

    sudo blkid $sdcard\1
출력 예시: /dev/sdb1: UUID="10f4b763-da3f-4a17-8bc3-babe754c76bd" TYPE="ext4"
    PARTLABEL="PARTLABEL" <span style=color:orange>PARTUUID="5f690f2e-9a29-4eb9-8780-8c47446f8054"</span> <br>
출력이 나오면 PARTUUID를 아래 명령어로 만든 파일에 입력해둔다. 추후에 bootloader 찾을때 사용되기때문<br><br>

    vim Linux_for_Tegra/bootloader/l4t-rootfs-uuid.txt_ext
#여기서 jetpack버전에 따라 달라지는데 뒤에 txt 인지 txt_ext인지는 버전에 따라 다르니 사용할 버전에 맞게 만들면된다. 

    sudo mount $sdcard\1 /mnt
    cd Linux_for_Tegra/rootfs/
    sudo tar -cpf - * | ( cd /mnt ; sudo tar -xpf - )
완료되면 sdcard를 sync작업하고 unmount 작업까지 해준다.<br>

    sync
    sudo umount /mnt

이렇게되면 sdcard는 jetpack 설치가 완료되었고 carrier board에 꼽고 키면 바로 부팅디스크로 작동하게 된다.<br>

### 3.  **Flash 준비 및 Flashing** <br><br>
SD카드를 연결한 뒤 jetson nano를 recovery 모드로 부팅한다. <br>
PC에서 1번 단계에서 준비해뒀던 BSP 파일 ( 덮어쓰기가 완료된 )에서 flashing을 진행한다. <br>

    sudo ./flash.sh jetson-xavier-nano external

완료되면 jetson이 재시작되고 모니터를 연결해 정상 작동하는지 확인한다. <br>

HADRON 640R 같은경우에는 /dev 를 확인했을때 정상 연결시에 video 장치가 2개가 잡혀야한다.(일반카메라와 열화상카메라) <br>

열화상카메라와 일반카메라는 내장프로그램인 cheese를 사용해서 확인해도 되지만 gstreamer를 통해 확인하는것이 편하다. <br>

일반카메라 명령어 <br>

    gst-launch-1.0 v4l2src device=/dev/video0 do-timestamp=true ! 'video/x-raw, width=1920, height=1080, framerate=30/1, format=UYVY' ! xvimagesink sync=false

열화상카메라 명령어 <br>

    gst-launch-1.0 v4l2src device=/dev/video1 ! 'video/x-raw, width=640, height=512, framerate=30/1' ! xvimagesink

ddddd