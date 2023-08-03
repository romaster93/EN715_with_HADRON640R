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



### 4.  **More setting for memory** <br><br>
최신 jetson 시리즈를 사용하는사람에게는 거의 해당되지 않는 내용이다. <br>
해당 step은 jetson nano를 사용하다보니 build 하거나 slam을 구동할 때 계속해서 memory 부족 현상이 발생하는데, 이를 해결해주기 위한 방법이다. <br>
필요한 사람만 참고하면 될 것 같다. <br>

jetson nano는 램이 4기가인데 이걸로는 요즘 작업이 거의 불가능하다. (특히 메모리를 많이 차지하는 시스템을 구동할때) <br>

2가지를 진행하는데 첫번째는 swap 메모리 공간을 넉넉하게 만들어주는것이고, 두번째는 jetson nano에서 사용하는 memory stack size를 늘려주는 작업을 진행한다.<br>

### 4.1  **Swap memory** <br><br>

본인은 sd card의 용량을 1tb를 사용하고 있기때문에 여유 공간이 매우 많은 상황이었다. <br>
따라서 swap 메모리를 무식하게 많이 만들어도 상관없었지만 그게 아닌 사용자들은 자신의 여유 공간에 맞게 설정하기 바란다. <br>

swap memory 활성화 여부 확인 <br>

    free -m    

swapon 명령으로 스왑 상태를 표시 <br>

    swapon -s  

dd 명령어를 통해서 swap file을 생성해준다. (나는 어짜피 공간이 남아돌아서 30G를 만들었는데 이러면 시간이 꽤 오래걸린다.) <br>

    sudo dd if=/dev/zero of=/var/swapfile bs=1G count=30   

mkswap 명령어를 통해서 초기화 진행<br>

    sudo mkswap /var/swapfile

root 사용자 권한 변경<br>

    sudo chmod 600 /var/swapfile

Jetson nano 부팅할때 자동으로 마운트 시켜주는 작업 진행 <br>

    sudo vim /etc/fstab

아래 문장을 파일 맨 아래에 추가한다. <br>

    /var/swapfile   /none   swap    swap    0 0

swapon 명령으로 활성화 진행 <br>

    sudo swapon /var/swapfile

free 명령으로 스왑이 확장된걸 확인 (만약에 확인이 안되면 재부팅 진행) <br>

    free -m 

swapon 명령으로 스왑의 상태를 확인해준다. <br>

    swapon -s


### 4.2  **Jetson nano memory stack size unlock** <br><br>

아무리 swap 메모리를 통해서 메모리 용량을 늘려줘도 설정값을 바꾸지 않으면 기존 내장 메모리만 사용하기 때문에 의미가 없다. <br>

따라서 jetson nano의 설정값을 바꿔줘야한다. <br>

    cd /etc/security
    sudo vim limits.conf

해당 파일에 들어가면 마지막 부분에 아래와 같은 parameter가 있는데 전부 unlimited로 변경해주면 된다. <br>

    nvidia hard stack unlimited
    nvidia soft stack unlimited
    ubuntu hard stack unlimited
    ubuntu soft stack unlimited
    root hard stack unlimited
    root soft stack unlimited




