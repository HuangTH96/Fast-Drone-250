<font size=6> **从零制作自主空中机器人** </font>

本文档是从视频教程[从零制作自主空中机器人](https://www.bilibili.com/video/BV1WZ4y167me?p=1)改编而来，用于适配实验室DIY的10寸机和5寸机。查看原教程[github仓库](https://github.com/ZJU-FAST-Lab/Fast-Drone-250)

<font color="#dd0000">安全事项</font>

四旋翼无人机具有较高的安全风险，请同学们严格遵守安全规范，不要在有人的室内或室外进行试验，对自己和他人的安全负责，本实验室完全免责。


- [第一章：动力套焊接](#第一章动力套焊接)
- [第二章：机载电脑与相机的连接](#第二章机载电脑与相机的连接)
- [第三章：飞控设置与试飞](#第三章飞控设置与试飞)
- [第四章：Ubuntu20.04的安装](#第四章ubuntu2004的安装)
- [第五章：机载电脑的环境配置](#第五章机载电脑的环境配置)
- [第六章：常用实验与调试软件的安装与使用](#第六章常用实验与调试软件的安装与使用)
- [第七章：Ego-Planner代码框架与参数介绍](#第七章ego-planner代码框架与参数介绍)
- [第八章：VINS的参数设置与外参标定](#第八章vins的参数设置与外参标定)
- [第九章：Ego-Planner的实验](#第九章ego-planner的实验)
- [Q&A 常见问题及解答](#qa-常见问题及解答)

## 第一章：动力套焊接
  查看[笔记](https://app.notion.com/p/2-10-3d2b7664a095805aa339c0923253d95e?v=2f6b7664a0958013a6a6000c337cbfa1&source=copy_link)
## 第二章：机载电脑与相机的连接
* 相机通过 usb3.0 和IntelNUC 通信，也就是要使用支持 usb3.0的数据线
* 飞控和机载电脑通过usb2ttl适配器通信：IntelNUC - FTDI 适配器 - Pixhawk（TELEM2）
  * 使用转接线连接TELEM2口和适配器的针脚 \
  ![接口说明](./Docs/telem2%20-%20ftdi.png)
* Linux按插入顺序，为USB设备分配名字，为了防止出错，为 USB Serial Port 指定名字
  * 查看设备ID `lsusb`，记录其中 pixhawk相关的口，如: Bus 003 Device 005: ID 26ac:0011
  * 更改UDEV规则文件
    ```
    cd /etc/udev/rules.d/
    sudo vim 98-pixhawk.rules
    ```
  * 为 pixhawk串口创建 sumlink `SUBSYSTEM=="tty", ATTRS{idVendor}=="26ac", ATTRS{idProduct}=="0011", SYMLINK+="ttyPixhawk"`

  * 重新加载udev规则，查看是否成功
    ```
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ls -la /dev/ttyPixhawk
    ```
    成功时应该显示 `lrwxrwxrwx 1 root root 7 ... /dev/ttyPixhawk -> ttyUSB0`
    
* 检查飞控和IntelNUC是否连接成功
  * 查看是否识别适配器 `ls /dev/ttyUSB*`，插上模块前后应该能看到有一个新的设备
  * 查看数据读取 `ls -la /dev/ttyUSB0`，如果显示 `crw-rw---- root dialout`， 需要确认当前用户在 dialout 组 `groups $USER`，如果没有`dialout`，则需要手动授权 `sudo usermod -a -G dialout $USER`，然后重新登录，或者临时，`sudo chmod 666 /dev/ttyPixhawk`
  * 确定串口有数据流入 `sudo cat /dev/ttyUSB0 | hexdump -C | head -30`，如果没有，需要在QGC中设置 `MAV_1_CONFIG` 应该设置成 `TELEM2`，reboot pixhawk后查看SER_TEL2_BAUD，应该和MAVROS波特率一致，`roscat mavros px4.launch` 查看MAVROS中fuc_url的波特率
  * 确认 MAVROS 启动参数
    ```
    roslaunch mavros px4.launch fcu_url:=/dev/ttyPixhawk:921600
    rostopic echo /mavros/state
    ``` 
    正常应该看到：
    ```
    connected: True
    armed: False
    mode: "MANUAL"
    ```
## 第三章：飞控设置与试飞
*记录介绍滤波和PIDs调节*
* 在QGC -> Analyze Tools -> MAVLink Console 中修改IMU发布频率
  ```
  nxh>ls && cd /fs/microsd && mkdir etc
  nxh>echo "mavlink stream -d /dev/ttyS3 -s ATTITUDE_QUATERNION -r 200" > /fs/microsd/etc/extras.txt
  nxh>echo "mavlink stream -d /dev/ttyS3 -s HIGHRES_IMU -r 200" >> /fs/microsd/etc/extras.txt
  ```
* 上电前请先用万用表通断档检测电源正负焊点是否短接，强烈建议第一次上电前先接一个[短路保护器](https://item.taobao.com/item.htm?spm=a230r.1.14.6.72b83b20uNbZk7&id=656973651729&ns=1&abbucket=19#detail)

*TODOs*:
* 滤波笔记
* PIDs调节笔记

## 第四章：Ubuntu20.04的安装

* 镜像站地址：`http://mirrors.aliyun.com/ubuntu-releases/20.04/` 下载 `ubuntu-20.04.4-desktop-amd64.iso`
* 烧录软件UltraISO官网：`https://cn.ultraiso.net/`
* 分区设置：
  * EFI系统分区（主分区）512M
  * 交换空间（逻辑分区）16000M（内存大小的两倍）
  * 挂载点`/`（主分区）剩余所有容量
  * <font color="#dd0000">笔记本上也需要安装ubuntu，推荐装20.04版本。虚拟机或双系统都可以，如果有长期学习打算推荐双系统</font>

## 第五章：机载电脑的环境配置
* ROS安装
  * `sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'`
  * `sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654`
  * `sudo apt update`
  * `sudo apt install ros-noetic-desktop-full`
  * `echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc`
  * <font color="#dd0000">建议没有ROS基础的同学先去B站学习古月老师的ROS入门教程</font>
* 测试ROS
  * 打开三个终端，分别输入
  * `roscore`
  * `rosrun turtlesim turtlesim_node`
  * `rosrun turtlesim turtle_teleop_key`
* realsense驱动安装
  * `sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key  F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key  F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE`
  * `sudo add-apt-repository "deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main" -u`
  * `sudo apt-get install librealsense2-dkms`
  * `sudo apt-get install librealsense2-utils`
  * `sudo apt-get install librealsense2-dev`
  * `sudo apt-get install librealsense2-dbg`
  * 测试：`realsense-viewer`
  * <font color="#dd0000">注意测试时左上角显示的USB必须是3.x，如果是2.x，可能是USB线是2.0的，或者插在了2.0的USB口上（3.0的线和口都是蓝色的）</font>
* 安装mavros
  * `sudo apt-get install ros-noetic-mavros`
  * `cd /opt/ros/noetic/lib/mavros`
  * `sudo ./install_geographiclib_datasets.sh`
* 安装ceres与glog与ddyanmic-reconfigure
  * `git clone git@github.com:HuangTH96/Fast-Drone-250.git`
  * `cd Fast-Drone-250`
  * 解压`3rd_party.zip`压缩包
  * 进入glog文件夹打开终端
  * `./autogen.sh && ./configure && make && sudo make install`
  * `sudo apt-get install liblapack-dev libsuitesparse-dev libcxsparse3 libgflags-dev libgoogle-glog-dev libgtest-dev`
  * 进入ceres文件夹打开终端
  * `mkdir build`
  * `cd build`
  * `cmake ..`
  * `sudo make -j4`
  * `sudo make install`
  * `sudo apt-get install ros-noetic-ddynamic-reconfigure`
* 下载ego-planner源码并编译
  * `cd ~/Fast-Drone-250`
  * `catkin_make`
  * `source devel/setup.bash`
  * `roslaunch ego_planner single_run_in_sim.launch`
  * 在Rviz内按下键盘G键，再单击鼠标左键以点选无人机目标点

## 第六章：常用实验与调试软件的安装与使用
* VScode：`sudo dpkg -i ***.deb`
* Terminator：`sudo apt install terminator`
* Plotjuggler：
  * `sudo apt install ros-noetic-plotjuggler`
  * `sudo apt install ros-noetic-plotjuggler-ros`
  * `rosrun plotjuggler plotjuggler`
* Net-tools：
  * `sudo apt install net-tools`
  * `ifconfig`
* ssh：
  * `sudo apt install openssh-server`
  * 在笔记本上：`ping 192.168.**.**`
  * `sudo gedit /etc/hosts`
  * 加上一行：`192.168.**.** fast-drone-10-1` 其中，`fast-drone-10-1` 为别名，取名规则为：`fast-drone-<10寸>-<1号机>`
  * `ping fast-drone-10-1`
  * `ssh intel@fast-drone-10-1`(`ssh 机载电脑用户名@别名`)

## 第七章：Ego-Planner代码框架与参数介绍
* `src/planner/plan_manage/launch/single_run_in_exp.launch`下的：
  * `map_size`：当你的地图大小较大时需要修改，注意目标点不要超过map_size/2
  * `fx/fy/cx/cy`：修改为你的深度相机的实际内参（下一课有讲怎么看）
  * `max_vel/max_acc`：修改以调整最大速度、加速度。速度建议先用0.5试飞，最大不要超过2.5，加速度不要超过6
  * `flight_type`：1代表rviz选点模式，2代表waypoints跟踪模式
* `src/planner/plan_manage/launch/advanced_param_exp.xml`下的：
  * `resolution`：代表栅格地图格点的分辨率，单位为米。越小则地图越精细，但越占内存。最小不要低于0.1
  * `obstacles_inflation`：代表障碍物膨胀大小，单位为米。建议至少设置为飞机半径（包括螺旋桨、桨保）的1.5倍以上，但不要超过`resolution`的4倍。如果飞机轴距较大，请相应改大`resolution`
* `src/realflight_modules/px4ctrl/config/ctrl_param_fpv.yaml`下的：
  * `mass`：修改为无人机的实际重量
  * `hover_percent`：修改为无人机的悬停油门，可以通过px4log查看，具体可以参考[文档](https://www.bookstack.cn/read/px4-user-guide/zh-log-flight_review.md) 也就是PX4中的 `MPC_THR_HOVER`
  * `gain/Kp,Kv`：即PID中的PI项，一般不用太大改动。如果发生超调，请适当调小。如果无人机响应较慢，请适当调大。
  * `rc_reverse`：这项使用乐迪AT9S的不用管。如果在第十一课的自动起飞中，发现飞机的飞行方向与摇杆方向相反，说明需要修改此项，把相反的通道对应的值改为true。其中throttle如果反了，实际实验中会比较危险，建议在起飞前就确认好，步骤为：
    * `roslaunch mavros px4.launch`
    * `rostopic echo /mavros/rc/in`
    * 打开遥控器，把遥控器油门从最低满满打到最高
    * 看echo出来的消息里哪项在缓慢变化（这项就是油门通道值），并观察它是不是由小变大
    * 如果是由小变大，则不需要修改throttle的rc_reverse，反之改为true
    * 其他通道同理
  
## 第八章：VINS的参数设置与外参标定
* 检查飞控mavros连接正常
  * `ls /dev/ttyPixhawk`
  * `sudo chmod 777 /dev/ttyPixhawk`，为串口附加权限
  * `roslaunch mavors px4.launch fuc_url:=/dev/ttyPixhawk:921600`
  * `rostopic hz /mavros/imu/data_raw`，确认飞控传输的imu频率在200hz左右
* 检查realsense驱动正常
  * `roslaunch realsense2_camera rs_camera.launch`
  * 进入远程桌面，`rqt_image_view`
  * 查看`/camera/infra1/image_rect_raw`,`/camera/infra2/image_rect_raw`,`/camera/depth/image_rect_raw`话题正常
* VINS参数设置
  * 进入`realflight_modules/VINS_Fusion/config/`
  * 驱动realsense后，`rostopic echo /camera/infra1/camera_info`，`rostopic echo /camera/infra2/camera_info`，分别把其中的K矩阵中的fx,fy,cx,cy填入`/src/realflight_modules/VINS-Fusion/config/left.yaml`和`/src/realflight_modules/VINS-Fusion/config/right.yaml`
  * 在home目录创建文件夹`mkdir ~/vins_output` (如果你的用户名不是intel，需要修改`/fast-drone-250.yaml`内的vins_out_path为你实际创建的文件夹的绝对路径)
  * 修改`/src/realflight_modules/VINS-Fusion/config/fast-drone-250.yaml`的`body_T_cam0`和`body_T_cam1`的`data`矩阵的第四列为你的无人机上的相机相对于飞控的实际外参，单位为米，顺序为x/y/z，第四项是1，不用改
  
* VINS外参精确自标定
  * `sh shfiles/rspx4.sh`
  * `rostopic echo /vins_fusion/imu_propagate`
  * 拿起飞机沿着场地<font color="#dd0000">尽量缓慢</font>地行走，场地内光照变化不要太大，灯光不要太暗，<font color="#dd0000">不要使用会频闪的光源</font>，尽量多放些杂物来增加VINS用于匹配的特征点
  * 把`vins_output/extrinsic_parameter.txt`里的内容替换到`/src/realflight_modules/VINS-Fusion/config/fast-drone-250.yaml`的`body_T_cam0`和`body_T_cam1`
  * 重复上述操作直到走几圈后VINS的里程计数据偏差收敛到满意值（一般在0.3米内）
* 建图模块验证
  * `sh shfiles/rspx4.sh`
  * `roslaunch ego_planner single_run_in_exp.launch`
  * 进入远程桌面 `roslaunch ego_planner rviz.launch` 查看在rviz中是否正确构建栅格地图

## 第九章：Ego-Planner的实验
* 自动起飞：
  * `sh shfiles/rspx4.sh`
  * `rostopic echo /vins_fusion/imu_propagate`
  * 拿起飞机进行缓慢的小范围晃动，放回原地后确认没有太大误差
  * 查看[笔记](https://app.notion.com/p/FastDrone250-px4ctrl-374b7664a095808f8d5bff7543ac9a18)，了解遥控器通道设置以及切换逻辑。将遥控器5通道拨到内侧，六通道拨到下侧，油门打到中位
  * `roslaunch px4ctrl run_ctrl.launch`
  * `sh shfiles/takeoff.sh`，如果飞机螺旋桨开始旋转，但无法起飞，说明`hover_percent`参数过小；如果飞机有明显飞过1米高，再下降的样子，说明`hover_percent`参数过大
  * 遥控器此时可以以类似大疆飞机的操作逻辑对无人机进行位置控制
  * 降落时把油门打到最低，等无人机降到地上后，把5通道拨到中间，左手杆打到左下角上锁
* Ego-Planner实验
  * 自动起飞
  * `roslaunch ego_planner single_run_in_exp.launch`
  * `sh shfiles/record.sh`
  * 进入远程桌面 `roslaunch ego_planner rviz.launch`
  * 按下G键加鼠标左键点选目标点使无人机飞行
* <font color="#dd0000">如果实验中遇到意外怎么办！！！</font>
  * `case 1`: VINS定位没有飘，但是规划不及时/建图不准确导致无人机规划出一条可能撞进障碍物的轨迹。如果飞手在飞机飞行过程中发现无人机可能会撞到障碍物，在撞上前把6通道拨回上侧，此时无人机会退出轨迹跟随模式，进入VINS悬停模式，在此时把无人机安全着陆即可
  * `case 2`：VINS定位飘了，表现为飞机大幅度颤抖/明显没有沿着正常轨迹走/快速上升/快速下降等等，此时拨6通道已经无济于事，必须把5通道拨回中位，使无人机完全退出程序控制，回到遥控器的stablized模式来操控降落
  * `case 3`：无人机已经撞到障碍物，并且还没掉到地上。此时先拨6通道，看看飞机能不能稳住，稳不住就拨5通道手动降落
  * `case 4`：无人机撞到障碍物并且炸到地上了：拨5通道立刻上锁，减少财产损失
  * `case 5`：**绝招** 反应不过来哪种case，或者飞机冲着非常危险的区域飞了，直接拨7通道紧急停桨。这样飞机会直接失去动力摔下来，对飞机机身破坏比较大，一般慢速情况下不建议。

## Q&A 常见问题及解答

	Q: 能不能用265+435来不跑vins？
	A: 可以，但265直出的里程计的速度估计有问题，可能导致控制不稳定。需要把265和imu做ekf融合。
	
	Q: 硬件清单中的xxx能不能更换？
	A: 请看视频番外一，讲解了大部分替换可能。
	   如果要换大轴距机架，请相应更换动力套及桨叶。pid参数也需要相应调整，相关内容自行查阅。
	   435相机可以换430相机。430更便宜但没有外壳，不好固定且容易炸坏。
	   电池不建议更换，因为课程的Q250机架刚刚好可以塞入2300mah 4S电池，不需要额外固定。更换电池需要自行解决电池放置问题。
	
	Q: 能不能用D435i自带的imu运行vins?
	A: 不行，因为435的imu噪声很大
	
	Q: 课程提供的v1.11.0固件有什么改动吗？必须使用这个固件吗？
	A: 没有任何改动，是直接从px4官方下载的。目前仅在该版本上测试通过了本套代码，且在v1.13上测试失败，表现为VINS会经常崩溃。其他飞控/其他版本固件没有测试，有需要的同学可以自行测试。
	
	Q: QGC内测试电机不转怎么办？
	A: 1. 检查电调是否支持dshot，不支持请自行查阅pwm电调校准方法。 
	   2.如果是使用V5+飞控或其他把模拟和数字输出分开的飞控（特点是输出口标号为A1~A4 M1~M4），如果要用Dshot协议，请插在A口上
	   3.使用holybro pixhawk4完全版飞控，飞控与分电板的插线请插在FMU PWM OUTPUT上，而非I/O PWM OUTPUT
	
	Q: 运行vins后报红字错误？
	A: 大概率是你改config后格式错误，照着报错去修改对应的config
	
	Q: 运行vins后报"VINS_RESULT_PATH not opened"?
	A: 在home目录创建`vins_output`文件夹(如果你的用户名不是fast-drone，需要修改config内的vins_out_path为你实际创建的文件夹的绝对路径)
	
	Q: 这台飞机的载重有多少？续航有多少？能飞多远？
	A: 不带额外负载起飞重量在1.1~1.2kg左右，最大起飞重量在1.8kg内，再大控制不稳且续航很短。
		不带负载续航约5分钟。
		能飞多远取决于你wifi的通讯质量，一般wifi顶多通讯100米。此外由于栅格地图直接开在内存内，如果地图范围设置过大，容易占满内存导致其他程序运行缓慢。一般不建议超过50米*50米。
		
	Q: 为什么要挡住D435的结构光？
	A: 结构光的意义在于使相机得到的深度图更准确，但双目图片上会显示出位置固定不变的点阵光斑，这对VIO的运行是不利的，所以需要关掉。
	
	Q: VINS飘怎么办？
	A: 1. 检查环境中是否有强反光物体（瓷砖、玻璃等）
	   2. 尽量缓慢地移动无人机，场景内不要有运动物体
	   3. 尽量准确地测量初始外参
	   4. 不要在运行vins的时候在远程桌面上运行rviz（会占用大量CPU资源），实在想开建议去配一下ROS多机，然后在笔记本上开
	   5. 检查环境中是否有频闪光源（肉眼无法看出，在realsense的单目画面中检查）
	
	Q: 我没有自稳模式无人机的飞行经验，身边也没有有经验的飞手，怎么办呢？
	A: 1. 有预算的情况下，建议购买一台耐摔的带保护圈的穿越机来练手，推荐的型号有mobula6,吉朗小金鱼85x,化骨龙racewhoop30等
	   2. 没啥预算的情况下，建议购买一个遥控器加密狗来把课程推荐的AT9S遥控器连接到电脑，然后在模拟器里练熟。模拟器推荐steam上的liftoff,免费的推荐free rider
	
	Q: 我想用NX做机载电脑，该另外做些什么？
	A: 原则上不建议小白用NX，会多很多麻烦事，本课程并不涉及，助教也没有时间去帮你看。
	   需要额外做下面的事：
	   1. 修改VINS为GPU版本的，因为NX的CPU算力很差，跑课程的CPU版VINS一定跑不动
	   2. 解决NX固定及供电问题
	   3. 解决realsense固定问题
	   4. 解决NX用小底板时接口不够的问题
	   5. 解决一系列arm和x86不兼容带来的问题
	   
	Q: 我按照教程修改sd卡里的etc/extra.txt后，IMU频率没有变成200hz怎么办？
	A: 可能原因是你的固件不支持这样修改，可以尝试在启动mavros后执行：
	rosrun mavros mavcmd long 511 105 5000 0 0 0 0 0 & sleep 1;
	rosrun mavros mavcmd long 511 31 5000 0 0 0 0 0 & sleep 1;
	

