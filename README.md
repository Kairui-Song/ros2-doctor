​机器人不动了，很多人的第一反应是：
"是不是控制算法错了？"
"是不是 PID 参数不对？"

先别猜。 按下面这条链路，从最上层逐层往下查，每一步拿到证据再走下一步。
一、先建立排障思维
机器人不工作，不要上来就看代码。按这条链查：

程序有没有启动？
        ↓
Node 存不存在？
        ↓
Controller 有没有 active？
        ↓
Topic 存不存在？
        ↓
有没有 Publisher？
        ↓
有没有 Subscriber？
        ↓
有没有数据流过？
        ↓
频率正常吗？/ QoS 匹配吗？
        ↓
整个通信链路通不通？

只有通信层全部正常，才允许讨论控制算法和硬件。

二、第 1 步：Node 有没有启动？
ros2 node list
例如：

/controller_manager
/joint_state_broadcaster
/robot_state_publisher
/teleop_node
如果你以为某个节点已经启动，但列表里根本没有它——停止往下查。Node 都没起来，Topic 自然不会通。

分支判断：




三、第 2 步：Controller 有没有 active？
这是 ROS2 Control 系统专属的一步，也是最容易漏掉的一层。

ros2 control list_controllers
例如：

joint
_state_broadcaster    active
diff_drive_controller      active
分支判断：




四、第 3 步：Topic 存不存在？
这一步只执行一次。 不要反复跑。

ros2 topic list
也可以带类型：

ros2 topic list -t
输出：

/diff_drive_controller/cmd_vel [geometry_msgs/msg/TwistStamped]
/joint_states [sensor_msgs/msg/JointState]
/imu [sensor_msgs/msg/Imu]
关键：不要假设速度指令 topic 叫 /cmd_vel。必须从实际输出里找。

五、第 4 步：谁在发？谁在收？
ros2 topic info /diff_drive_controller/cmd_vel

输出：

Type: geometry_msgs/msg/TwistStamped
Publisher count: 0
Subscription count: 1

分支判断：

结果	含义	下一步
Publisher: 0	上游没在发，故障点已定位	停止，检查遥控/导航节点是否启动
Subscriber: 0	controller 没订阅	检查 controller 配置里的 topic 名
都有	继续	进入第 5 步
注意：一旦执行过 topic info，不要用 -v 重复执行，除非走到第 7 步查 QoS。

六、第 5 步：有没有数据流过？
ros2 topic echo /diff_drive_controller/cmd_vel



这是最容易走错的分叉点："没有数据"不等于"频率为 0"。没数据时跑 hz 毫无意义，应该直接查 QoS。

七、第 6 步：频率正常吗？
ros2 topic hz /diff_drive_controller/cmd_vel
输出：

average rate: 0.2
min: 0.100s max: 8.500s std dev: 3.20s

判断：



八、第 7 步：QoS 匹配吗？
echo 无输出时，这是最可能的元凶。

ros2 topic info /diff_drive_controller/cmd_vel -v
输出：

Publisher:
  Node name: teleop_node
  Reliability: BEST_EFFORT
Subscriber:
  Node name: diff_drive_controller
  Reliability: RELIABLE

判断：



QoS 不匹配是 ROS2 里"topic 存在但没数据"最常见的原因。

九、第 8 步：看整个系统结构
rqt_graph
它把 Node 和 Topic 的连接关系画出来：

teleop_node
      │
      │ /diff_drive_controller/cmd_vel
      ↓
diff_drive_controller
      │
      ↓
hardware_interface
判断：



十、ROS2 排障地图
             出问题
                │
                ↓
        ros2 node list
                │
          Node 在吗？
          /        \
         否         是
         ↓          ↓
      查启动   list_controllers
                     │
                Controller active？
                 /            \
                否             是
                ↓              ↓
            激活它       ros2 topic list
                              │
                          Topic 在吗？
                          /        \
                         否        是
                         ↓         ↓
                     查代码   topic info
                                   │
                             Publisher？
                             Subscriber？
                              /    |    \
                            无P    无S   都有
                             ↓     ↓     ↓
                           停    停  topic echo
                                          │
                                     有数据？
                                      /    \
                                     否     是
                                     ↓      ↓
                                QoS检查  topic hz
                                          │
                                       频率？
                                       /    \
                                     异常   正常
                                      ↓      ↓
                                     停   rqt_graph
                                            │
                                       链路完整？
                                        /    \
                                       否     是
                                       ↓      ↓
                                      停  允许查算法

十一、诊断结束格式
定位到故障点后，按这个格式输出：

十二、这套流程已经变成 Skill
这套诊断逻辑被完整写成了一份 SKILL.md，上传到 Claude 后，只要你说一句 "我的 ROS2 机器人不动了"，它就会：
