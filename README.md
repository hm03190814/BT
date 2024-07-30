<h1 align="center">BehaviorTree</h1>

# Link


- 官方文档 https://www.behaviortree.dev/docs/intro
- groot https://www.behaviortree.dev/groot
- C++库 https://github.com/BehaviorTree/BehaviorTree.CPP
- ROS库 https://wiki.ros.org/behavior_tree
- Python库 https://github.com/futureneer/beetree
- 知乎行为树教程 https://www.zhihu.com/column/c_1572932842551775232

# install BT4

- install

```shell
# 安装各项依赖
sudo apt install -y libboost-dev libboost-coroutine-dev libzmq3-dev
sudo apt install -y qtbase5-dev libqt5svg5-dev libzmq3-dev libdw-dev

# clone BT4 from github
cd /tmp && git clone https://github.com/BehaviorTree/BehaviorTree.CPP.git

# 编译安装
cd /tmp/BehaviorTree.CPP/ && mkdir build && cd build && cmake .. 
make -j16 && sudo make install
```

-  安装完成后应该有以下内容

```shell
/usr/local/lib/libbehaviortree_cpp.so
/usr/local/include/behaviortree_cpp
```

- test

```c++
cd /tmp/BehaviorTree.CPP/build/examples && ./t01_first_tree_static
```

- some_link

[CSDN - 专栏 - BT](https://blog.csdn.net/whahu1989/category_10717968.html?spm=1001.2014.3001.5482)   

[ROS2中的行为树 BehaviorTree](https://cloud.tencent.com/developer/article/2030718) 

[ROS2行为树（C++行为树）BehaviorTree.CPP完全图形化开发，完美支持ROS2话题通信](https://blog.csdn.net/m0_63671696/article/details/131945756)  

[ROS2中的行为树 BehaviorTree](https://hermit.blog.csdn.net/article/details/125492668) 

[行为树入门：ROS2 BehaviorTree.CPP Groot2安装与Groot2使用（有例程）（1）](https://blog.csdn.net/LW_12345/article/details/136078563) 

# install Groot2

- download_link https://www.behaviortree.dev/groot/ 

```shell
# 切换到下载路径，给安装脚本加权限
chmod  +x Groot2-v*.run && ./Groot2-v*.run 
# 安装路径填写 /opt/Groot2 (不用 mkdir /opt/Groot2)
# 配置环境变量
echo "alias groot2=/opt/Groot2/bin/groot2" >> ~/.bashrc
```

# postscript

- 2024.06.22 还差groot使用和集成ROS
- 2024.07.30 完善了install板块，更新了README.md

