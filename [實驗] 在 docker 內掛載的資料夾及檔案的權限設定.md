# [實驗] 在 docker 內掛載的資料夾及檔案的權限設定
因為 docker volume 只能設定 read-only 跟 read-write\
為了設定 noexec 有了以下實驗\
本篇記錄 `docker run -v` 時, 原始資料夾不同 mount 參數對容器內檔案權限設定的影響
## 步驟紀錄
首先建立一個資料夾 ori 並建立一個可執行的 script
```
$ mkdir ori &&\
echo "echo yee" > ori/script &&\
chmod +x ori/script
$ tree .
.
└── ori
    └── script

1 directory, 1 file
```
接著建立四個資料夾\
分別對應 mount -o 的 ro, noexec, nosuid, nodev 選項\
並將 ori 以 bind mount 的方式以對應的選項 mount 到個別資料夾
```
$ mkdir ro noexec nosuid nodev
$ sudo mount -o bind,ro ori ro
$ sudo mount -o bind,noexec ori noexec
$ sudo mount -o bind,nosuid ori nosuid
$ sudo mount -o bind,nodev ori nodev
```
確認 mount 權限設定
```
$ mount | grep /mount
/dev/sdd on /.../mount/ro type ext4 (ro,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /.../mount/noexec type ext4 (rw,noexec,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /.../mount/nosuid type ext4 (rw,nosuid,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /.../mount/nodev type ext4 (rw,nodev,relatime,discard,errors=remount-ro,data=ordered)
```
建立 container 將 ro, noexec, nosuid, nodev 四個資料夾掛載進去\
並以上一步的命令在容器中確認權限設定
```
$ docker run --rm -v /.../mount/ro:/root/mnt/ro \
> -v /.../mount/noexec:/root/mnt/noexec \
> -v /.../mount/nosuid:/root/mnt/nosuid \
> -v /.../mount/nodev:/root/mnt/nodev \
> alpine mount | grep /root/mnt
/dev/sdd on /root/mnt/noexec type ext4 (rw,noexec,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /root/mnt/nosuid type ext4 (rw,nosuid,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /root/mnt/nodev type ext4 (rw,nodev,relatime,discard,errors=remount-ro,data=ordered)
/dev/sdd on /root/mnt/ro type ext4 (ro,relatime,discard,errors=remount-ro,data=ordered)
```
不過個別可行不代表混在一起也可以\
全部混在一起~~做撒尿牛丸~~再測試一次
```
$ mkdir nnn
$ sudo mount -o bind,ro,nodev,noexec,nosuid ori nnn
$ mount | grep nnn
/dev/sdd on /.../mount/nnn type ext4 (ro,nosuid,nodev,noexec,relatime,discard,errors=remount-ro,data=ordered)
$ docker run --rm -v /home/ruei/docker/mount/nnn:/root/mnt/nnn alpine mount | grep /root/mnt
/dev/sdd on /root/mnt/nnn type ext4 (ro,nosuid,nodev,noexec,relatime,discard,errors=remount-ro,data=ordered)
```
實驗結束
## takeaway
1. 要設定 noexec, nosuid, nodev 先 bind mount 再 mount 給容器
## WTF
```
$ mount | grep nnn
/dev/sdd on /.../mount/nnn type ext4 (ro,nosuid,nodev,noexec,relatime,discard,errors=remount-ro,data=ordered)
$ docker run --rm -v /home/ruei/docker/mount/nnn:/root/mnt/nnn:ro alpine sh -c /root/mnt/nnn/script
yee
```
