# 这是一个用来记录cawa数据集的处理过程的record
## 处理脚本及其作用
## gee数据
gee上下载数据到google drive，然后使用google drive for desktop。
cmd输入 ： robocopy "G:\我的云端硬盘\CAWa_LrgTiles_2018_Fixed""E:\zhongyashuju\CAWa_cropType\Sentinel2_Images\Downloads" *.tif /j /xo /r:3 /w:5

然后接可以看到数据在下载，这样的下载方式可以避免大量数据在网页下载时导致的网页超时从而下载失败。