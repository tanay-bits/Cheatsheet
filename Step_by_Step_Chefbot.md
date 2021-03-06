# Create Your Own Chefbot

This tutorial will show you how chefbot was created.
Chefbot is created according to the design of turtlebot.
You could find more information from the book "Learning Robotics Using Python"

## Prerequisites

### Install Turtlebot Packages

We have two methods to install Turtlebot: **From Source** or **apt Package Manager**

#### From Source

Install all dependencies(**ros-indigo-rocon** and **ros-indigo-kobuki**) using **synaptic**
```python
# open synaptic package manager
$ sudo synaptic

# search keyword ros-indigo-rocon, select all items for installation, and apply

# search keyword ros-indigo-kobuki, select all items for installation, and apply
```

Get/Install Turtlebot packages from source
```python
# create folder for turtlebot
$ mkdir ~/turtlebot

# go into the directory
$ cd ~/turtlebot

# download the latest source code of turtlebot
$ wstool init src -j5 https://raw.github.com/yujinrobot/yujin_tools/master/rosinstalls/indigo/turtlebot.rosinstall

# install all the dependencies of the source code
$ rosdep install --from-paths src -i -y

# build the source code
$ catkin_make

# to access turtlebot packages from all terminals
$ echo "source ~/turtlebot/devel/setup.bash" >> ~/.bashrc

#  execute the .bashrc file
$ source ~/.bashrc
```
#### Using apt Package Manager
This will install all the necessary packages for turtlebot
```python
$ sudo apt-get insatll ros-indigo-turtlebot ros-indigo-turtlebot-apps ros-indigo-turtlebot-interactions ros-indigo-turtlebot-simulator ros-indigo-kobuki-ftdi ros-indigo-rocon-remocon

```

#### Test Out Turtlebot
```python
# Bring up the Turtlebot simulation using ROS and Gazebo
$ roslaunch turtlebot_gazebo turtlebot_playground.launch

# On another terminal, control Turtlebot using keyboard
$ roslaunch turtlebot_teleop keyboard_teleop.launch

```

### Create Custom Robot by Cloning Turtlebot
#### Download Chefbot Source Code
```python
# we will put chefbot package in the home directory.
# You can put anywhere you want since we did not launch file from this
$ cd

# clone the complete ROS packages of the Chefbot
$ git clone https://github.com/qboticslabs/Chefbot_ROS_pkgs.git

# create chefbot folder to let us put all the packages in it
$ cd ~/catkin_ws/src
$ mkdir chefbot
$ cd chefbot
```
#### Create chefbot_gazebo package
```python
# go to chefbot directory
$ cd ~/catkin_ws/src/chefbot

# create gazebo package for chefbot
$ catkin_create_pkg chefbot_gazebo depthimage_to_laserscan diagnostic_aggregator gazebo_ros kobuki_gazebo_plugins robot_pose_ekf robot_state_publisher xacro yocs_cmd_vel_mux create_gazebo_plugins create_description create_driver create_node

** Remember to copy launch folder and world folder from the chefbot source package into chefbot_gazebo package **
# $ cp ~/Chefbot_ROS_pkgs/chefbot/chefbot_simulator/chefbot_gazebo/launch ~/catkin_ws/src/chefbot/chefbot_gazebo/launch
# $ cp ~/Chefbot_ROS_pkgs/chefbot/chefbot_simulator/chefbot_gazebo/worlds ~/catkin_ws/src/chefbot/chefbot_gazebo/worlds
```

#### Create chefbot_description package
```python
# go to chefbot directory
$ cd ~/catkin_ws/src/chefbot

# create description package for chefbot
$ catkin_create_pkg chefbot_description urdf xacro kobuki_description create_description

** remember to copy meshes, urdf, robots **
```

#### Test Out Chefbot
```python
# Build the package
$ cd ~/catkin_ws
$ catkin_make

# launch Chefbot simulation in ROS and Gazebo
$ roslaunch chefbot_gazebo chefbot_playground.launch
```


## How the Chefbot Constructs
Since we clone the files from chefbot and turtlebot packages, we will try to explain the code following how the program been called.
To do so, we will create packages to build chefbot from scratch. There will have some minor adjustment with the joints and links

### Setup
We are going to name the robot myturtle
```python
# Create a folder for our robot packages
$ cd ~/catkin_ws/src
$ mkdir myturtle
$ cd myturtle
```

Gazebo Package
```python
# create gazebo package for myturtle
$ catkin_create_pkg myturtle_gazebo depthimage_to_laserscan diagnostic_aggregator gazebo_ros kobuki_gazebo_plugins robot_pose_ekf robot_state_publisher xacro yocs_cmd_vel_mux create_gazebo_plugins create_description create_driver create_node

# create launch and worlds folders
$ cd myturtle_gazebo
$ mkdir launch worlds

# create includes folder inside launch
$ mkdir -p launch/includes
```

Description Package
```python
$ cd ~/catkin_ws/src/myturtle

# create description package for myturtle
$ catkin_create_pkg myturtle_description urdf xacro kobuki_description create_description

# create meshes, robots, urdf directories
$ cd myturtle_description
$ mkdir meshes robots, urdf
```

Build myturtle packages
```python
$ cd ~/catkin_ws
$ catkin_make
```

### Launch a Empty World in Gazebo
Create a launch file named myturtle_empty_world.launch in launch folder
```python
# Since we use atom as the editor, we could create a file by following command
$ cd ~/catkin_ws/src/myturtle/launch
$ atom myturtle_empty_world.launch
```

We will load a empty_world from gazebo_ros package  
```xml
<!-- myturtle_empty_world.launch -->
<launch>
    <!-- load an empty world -->
    <include file="$(find gazebo_ros)/launch/empty_world.launch">
      <arg name="use_sim_time" value="true"/>
      <arg name="debug" value="false"/>
    </include>
</launch>
```

Launch the empty world, we will call this command to see what the robot looks like later on.
```python
$ roslaunch myturtle_gazebo myturtle_empty_world.launch
```

You should see a empty world in Gazebo
<br></br>
![Empty World](images/empty_world.png)

### Build MyTurtle From Scratch

#### Build MyTurtle Base
Even though our robot is not that complicated, we should still break our robot into several files
```
The file hierarchy is going to look like this in this stage
~/catkin_ws/src/myturtle
  /myturtle_gazebo
    /launch
      /include
        myturtle_base.launch.xml
      myturtle_empty_world.launch
    CMakeList.txt
    package.xml
  /myturtle_description
    /meshes
    /robots
      myturtle_circles_kinect.xacro
    /urdf
      myturtle_base.urdf.xacro
      myturtle_base_gazebo.urdf.xacro
      myturtle_library.urdf.xacro
    CMakeList.txt
    package.xml
```

##### Load MyTurtle Base in myturtle_empty_world.launch
We want to load the most basic frame of MyTurtle base into the launch file.
To let people to understand our robot structure without going through all the code,
we break down the program to several parts
```xml
<!-- myturtle_empty_world.launch -->
<launch>
  <!-- arguments that make loading different sensors easier -->
  <arg name="base"      value="$(optenv MYTURTLE_BASE myturtle_base)"/>
  <arg name="stacks"    value="$(optenv MYTURTLE_STACKS circles)"/>  <!-- circles -->
  <arg name="3d_sensor" value="$(optenv MYTURTLE_3D_SENSOR kinect)"/>  <!-- kinect, asus_xtion_pro -->

  <!-- load an empty world -->
  <include> ... </include>

  <!-- load myturtle with base, stacks shape, and 3d sensor according to specified arguments above -->
  <include file="$(find myturtle_gazebo)/launch/includes/$(arg base).launch.xml">
    <arg name="base" value="$(arg base)"/>
    <arg name="stacks" value="$(arg stacks)"/>
    <arg name="3d_sensor" value="$(arg 3d_sensor)"/>
  </include>
</launch>
```

myturtle_empty_world.launch loads the robot model by including myturtle_base.lacunch.xml, and
myturtle_base.launch.xml loads myturtle_circles_kinect.urdf.xacro as spawn model
```xml
<!-- myturtle_base.launch.xml -->
<launch>
  <!-- arguments that we passed in from myturtle_empty_world.launch -->
  <arg name="base"/>
  <arg name="stacks"/>
  <arg name="3d_sensor"/>

  <!-- load the urdf file using arguments that passed in -->
  <arg name="urdf_file" default="$(find xacro)/xacro.py '$(find myturtle_description)/robots/$(find myturtle_description)/robots/myturtle_$(arg stacks)_$(arg 3d_sensor).urdf.xacro' " />
  <!-- set parameter for the spawn_model node with argument create above(urdf_file) -->
  <param name="robot_description" command="$(arg urdf_file)" />
  <!-- load the robot description onto spawn model -->
  <node name="spawn_turtlebot_model" pkg="gazebo_ros" type="spawn_model"
        args="$(optenv ROBOT_INITIAL_POSE) -unpause -urdf -param robot_description -model mobile_base" />
</launch>
```

Since our robot is going to get really messy, we want to create file to collect all the urdf/xacro files
Therefore, we includes myturtle_library.urdf.xacro in myturtle_base_circles_kinect.urdf.xacro to access all the components(xacro:macros) of our robot model
```xml
<!-- myturtle_base_circles_kinect.urdf.xacro -->
<?xml version="1.0"?>
<!--
    Base: kobuki
    Stacks: hexagons
    3d Sensor: kinect
 -->
<robot name="turtlebot" xmlns:xacro="http://ros.org/wiki/xacro">
  <!-- myturtle_library.urdf.xacro load myturtle model, general meshes, and kinect sensor-->
  <xacro:include filename="$(find myturtle_description)/urdf/myturtle_library.urdf.xacro" />
</robot>
```

In this stage, we only need to include one urdf file to access the basic robot model
```xml
<!-- myturtle_library.urdf.xacro -->
<?xml version="1.0"?>
<robot name="myturtle" xmlns:xacro="http://ros.org/wiki/xacro">
  <!-- MyTurtle Base -->
  <xacro:include filename="$(find myturtle_description)/urdf/myturtle_base.urdf.xacro" />
</robot>
```

myturtle_base.urdf.xacro is used to describe how to robot should look like.
We will create a "kobuki" macro for the robot and load the robot base inside this macro first.
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0" ?>
<robot name="myturtle" xmlns:xacro="httpL//ros.org/wiki/xacro">

  <!--
    This is some variable that we might use later on. If we have too many variables
    that make the program unorganized, we will create a files just for variables.
    -->
  <xacro:property name="M_PI" value="3.1415926535897931" />

  <!-- custom variables -->
  <xacro:property name="BASE_OFFSET_GND" value="0.1" />
  <xacro:property name="BASE_VISUAL_OFFSET_JOINT" value="-0.045" />
  <xacro:property name="BASE_COLLISION_HEIGHT" value="0.1" />

  <xacro:macro name="kobuki">
    <!-- root link -->
    <link name="base_footprint"/>

    <!-- joint for base_footprint and base_link(root <- base_link) -->
    <joint name="base_joint" type="fixed">
        <origin xyz="0 0 ${BASE_OFFSET_GND}" rpy="0 0 0" />
        <parent link="base_footprint" />
        <child link="base_link" />
    </joint>

    <link name="base_link">
      <visual>
        <!-- uncomment the code below to see how the collision model looks like -->

        <!-- <geometry>
          <cylinder length="${BASE_COLLISION_HEIGHT}" radius="0.178"/>
        </geometry>
        <origin xyz="0.0 0 ${BASE_COLLISION_HEIGHT/2}" rpy="0 0 0"/> -->

        <!-- myturtle base, we use chefbot's base 3D model -->
        <geometry>
          <!-- new mesh -->
          <mesh filename="package://chefbot_description/meshes/base_plate.dae" />
        </geometry>
        <origin xyz="0.001 0 ${BASE_VISUAL_OFFSET_JOINT}" rpy="0 0 ${M_PI/2}"/>
      </visual>

      <collision>
        <geometry>
          <cylinder length="${BASE_COLLISION_HEIGHT}" radius="0.178"/>
        </geometry>
        <origin xyz="0.0 0 ${BASE_COLLISION_HEIGHT/2}" rpy="0 0 0"/>
      </collision>

      <inertial>
        <!-- COM experimentally determined -->
        <origin xyz="0.01 0 0"/>
        <mass value="2.4"/> <!-- 2.4/2.6 kg for small/big battery pack -->
        <!-- Kobuki's inertia tensor is approximated by a cylinder with homogeneous mass distribution
             More details: http://en.wikipedia.org/wiki/List_of_moment_of_inertia_tensors
             m = 2.4 kg; h = 0.09 m; r = 0.175 m
             ixx = 1/12 * m * (3 * r^2 + h^2)
             iyy = 1/12 * m * (3 * r^2 + h^2)
             izz = 1/2 * m * r^2
          -->
        <inertia ixx="0.019995" ixy="0.0" ixz="0.0"
                 iyy="0.019995" iyz="0.0"
                 izz="0.03675" />
       </inertial>
    </link>
  </xacro:macro>
</robot>
```
Because we use macro to wrap around the robot model, we need to accessed by calling the macro(otherwise, the model won't be accessed).
```xml
<!-- myturtle_circles_kinect.urdf.xacro -->
<?xml version="1.0"?>
<robot name="turtlebot" xmlns:xacro="http://ros.org/wiki/xacro">
  <!-- myturtle_library.urdf.xacro load myturtle model, general meshes, and kinect sensor-->
  <xacro:include filename="$(find myturtle_description)/urdf/myturtle_library.urdf.xacro" />
  <!-- we simply just use the macro's name for the robot model as a element -->
  <kobuki/>
</robot>
```
The Base should look like this
<br></br>
![Myturtle Frame](images/myturtle_frame.png)

###### Optional Step To Help You Understand
If you want to see how the collision model looks like, we use the geometry and origin of collision model into visual.
uncomment the code in visual tag and comment the robot model(.dae) we loaded in previous section.
<br></br>
Remember to change back to previous step after you done seeing the collision model since we don't need to see collision mode most of the time.
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0" ?>
<robot name="myturtle" xmlns:xacro="httpL//ros.org/wiki/xacro">

  ...

  <link name="base_link">
    <visual>
      <!-- uncomment the code below to see how the collision model looks like -->
      <geometry>
        <cylinder length="${BASE_COLLISION_HEIGHT}" radius="0.178"/>
      </geometry>
      <origin xyz="0.0 0 ${BASE_COLLISION_HEIGHT/2}" rpy="0 0 0"/>

      <!-- myturtle base, we use chefbot's base 3D model -->
      <!--
      <geometry> ... </geometry>
      <origin />
      -->
    </visual>
    <collision> ... </collision>
    <inertial> ... </inertial>
  </link>
</robot>
```
We can see that the collision model of the base is just a simple cylinder and not as tall as the actual base model. This is because we want to use simple shape for collision model to optimize the simulation speed
<br></br>
![MyTurtle Frame Collision](images/chefbot_frame_collision.png)

##### Give MyTurtle Wheels
We want to add left and right wheels to the robot. To do so, we approximate where the wheels should be based on the base plate. Since the given wheel model from chefbot is not accurate, we recreate our wheel and the measure the location of the motor and place our wheel at the right location. We can approximate the size and location of the wheel because we don't need high precision in this case.
<br></br>
We change the values of origin's z-value, cylinder's radius of collision and visual from 0.035 to 0.04 meters
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  ...
  <!-- custom variables -->
  ...
  <xacro:property name="WHEEL_OFFSET" value="${0.03 - BASE_VISUAL_OFFSET_JOINT}" /> <!-- WHEEL_OFFSET + BASE_VISUAL_OFFSET_JOINT = 0.03 -->
  <xacro:property name="WHEEL_RADIUS" value="0.035" />
  <xacro:property name="WHEEL_VISUAL_OFFSET" value="0.02" />

  <xacro:macro name="kobuki">
    ...
    <link name="base_link"> ... </link>

    <joint name="wheel_left_joint" type="continuous">
      <parent link="base_link"/>
      <child link="myturtle_wheel_left_link"/>
      <origin xyz="0 ${0.28/2} ${WHEEL_OFFSET+BASE_VISUAL_OFFSET_JOINT}" rpy="${-M_PI/2} 0 0"/>
      <axis xyz="0 0 1"/>
    </joint>
    <link name="myturtle_wheel_left_link">
      <visual>
        <geometry>
          <cylinder length="0.0206" radius="${WHEEL_RADIUS}"/>
        </geometry>
        <origin xyz="0 0 0" rpy="0 0 0"/>
      </visual>
      <collision>
        <geometry>
          <cylinder length="0.0206" radius="${WHEEL_RADIUS}"/>
        </geometry>
        <origin rpy="0 0 0" xyz="0 0 0"/>
      </collision>
      <inertial>
        <mass value="0.01" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.001" ixy="0.0" ixz="0.0"
                 iyy="0.001" iyz="0.0"
                 izz="0.001" />
      </inertial>
    </link>

    <joint name="wheel_right_joint" type="continuous">
      <parent link="base_link"/>
      <child link="myturtle_wheel_right_link"/>
      <origin xyz="0 -${0.28/2} ${WHEEL_OFFSET+BASE_VISUAL_OFFSET_JOINT}" rpy="${-M_PI/2} 0 0"/>
      <axis xyz="0 0 1"/>
    </joint>
    <link name="myturtle_wheel_right_link">
      <visual>
        <geometry>
          <cylinder length="0.0206" radius="${WHEEL_RADIUS}"/>
        </geometry>
        <origin xyz="0 0 0" rpy="0 0 0"/>
      </visual>
      <collision>
        <geometry>
          <cylinder length="0.0206" radius="${WHEEL_RADIUS}"/>
        </geometry>
        <origin rpy="0 0 0" xyz="0 0 0"/>
      </collision>
      <inertial>
        <mass value="0.01" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.001" ixy="0.0" ixz="0.0"
                 iyy="0.001" iyz="0.0"
                 izz="0.001" />
      </inertial>
    </link>
  </xacro:macro>
</robot>
```
After launching our program, we should see the robot with two wheels.
<br></br>
![MyTurtle with wheels](images/myturtle_with_wheels.png)

##### Setup Joints of Two Caster Wheels
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  <!-- custom variables -->
  ...
  <xacro:property name="CASTER_RADIUS" value="${WHEEL_RADIUS - 0.0305}" /> <!-- CASTER_RADIUS = WHEEL_RADIUS - 0.0305 -->

  ...
  <link name="myturtle_wheel_right_link"> ... </link>

  <!-- Joint between myturtle base link and myturtle front caster wheel -->
  <!-- Since caster wheel doesn't move in a particular direction, we set the joint to fixed -->
  <joint name="caster_front_joint" type="fixed">
    <parent link="base_link"/>
    <child link="myturtle_caster_front_link"/>
    <origin xyz="0.115 0.0 0" rpy="${-M_PI/2} 0 0"/>
  </joint>
  <link name="myturtle_caster_front_link">
    <!-- Uncomment the visual will let you see how the caster wheel collision looks like -->
    <!-- <visual>
      <geometry>
        <cylinder length="0.0176" radius="${CASTER_RADIUS}"/>
      </geometry>
      <origin rpy="0 0 0" xyz="0 0 0"/>
    </visual> -->
    <collision>
      <geometry>
        <cylinder length="0.0176" radius="${CASTER_RADIUS}"/>
      </geometry>
      <origin rpy="0 0 0" xyz="0 0 0"/>
    </collision>
    <inertial>
      <mass value="0.01" />
      <origin xyz="0 0 0" />
      <inertia ixx="0.001" ixy="0.0" ixz="0.0"
               iyy="0.001" iyz="0.0"
               izz="0.001" />
    </inertial>
  </link>


  <!-- Joint between myturtle base link and myturtle front caster wheel -->
  <joint name="myturtle_caster_back_joint" type="fixed">
    <parent link="base_link"/>
    <child link="myturtle_caster_back_link"/>
    <origin xyz="-0.135 0.0 0" rpy="${-M_PI/2} 0 0"/>
  </joint>
  <link name="myturtle_caster_back_link">
    <!-- Uncomment the visual can let you see how the caster wheel collision looks like -->
    <!-- <visual>
      <geometry>
        <cylinder length="0.0176" radius="${CASTER_RADIUS}"/>
      </geometry>
      <origin rpy="0 0 0" xyz="0 0 0"/>
    </visual> -->
    <collision>
      <geometry>
        <cylinder length="0.0176" radius="${CASTER_RADIUS}"/>
      </geometry>
      <origin rpy="0 0 0" xyz="0 0 0"/>
    </collision>
    <inertial>
      <mass value="0.01" />
      <origin xyz="0 0 0" />
      <inertia ixx="0.001" ixy="0.0" ixz="0.0"
               iyy="0.001" iyz="0.0"
               izz="0.001" />
    </inertial>
  </link>
</robot>
```
You can see that adding the joint of collision balanced the robot. The robot wasn't level because it doesn't have the support of front and caster wheel
<br></br>
![Caster Wheel Collision](images/myturtle_caster_collision.png)

Uncomment the visual section of code will let you see how the collision of caster wheels looks like
<br></br>
![Caster Wheel Visual of Collision](images/myturtle_caster_collision_visual.png)


##### Add Joints and Links for Gyro and Cliff Sensors
We will define the joint and links for gyro and cliff Sensors. Later on, we will defined sensors' properties for gazebo
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  ...
  <xacro:macro name="kobuki">
    ...
    <!-- Kobuki's sensors -->
    <joint name="gyro_joint" type="fixed">
      <axis xyz="0 1 0"/>
      <origin xyz="0.056 0.062 0.0202" rpy="0 0 0"/>
      <parent link="base_link"/>
      <child link="gyro_link"/>
    </joint>
    <link name="gyro_link">
      <inertial>
        <mass value="0.001"/>
        <origin xyz="0 0 0" rpy="0 0 0"/>
        <inertia ixx="0.0001" ixy="0" ixz="0"
                 iyy="0.000001" iyz="0"
                 izz="0.0001"/>
      </inertial>
    </link>

    <joint name="cliff_sensor_left_joint" type="fixed">
      <origin xyz="0.08734 0.13601 0.0214" rpy="0 ${M_PI/2} 0" />
      <parent link="base_link"/>
      <child link="cliff_sensor_left_link" />
    </joint>
    <link name="cliff_sensor_left_link">
      <inertial>
        <mass value="0.0001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
    </link>

    <joint name="cliff_sensor_right_joint" type="fixed">
      <origin xyz="0.085 -0.13601 0.0214" rpy="0 ${M_PI/2} 0" />
      <parent link="base_link"/>
      <child link="cliff_sensor_right_link" />
    </joint>
    <link name="cliff_sensor_right_link">
      <inertial>
        <mass value="0.0001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
    </link>

    <joint name="cliff_sensor_front_joint" type="fixed">
      <origin xyz="0.156 0.00 0.0214" rpy="0 ${M_PI/2} 0" />
      <parent link="base_link"/>
      <child link="cliff_sensor_front_link" />
    </joint>
    <link name="cliff_sensor_front_link">
      <inertial>
        <mass value="0.0001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
    </link>
  </xacro:macro>
</robot>
```

##### Adding Gazebo Properties for MyTurtle Base
We need to create a file myturtle_base_gazebo.urdf.xacro in urdf folder and include it in myturtle_base.urdf.xacro
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  <xacro:include name="$(find myturtle_description)/urdf/myturtle_base_gazebo.urdf.xacro"/>
  ...
</robot>
```

Let's take a look what gazebo properties we need to add to our sensors
```xml
<?xml version="1.0"?>

<robot name="kobuki_sim" xmlns:xacro="http://ros.org/wiki/xacro">
  <xacro:macro name="kobuki_sim">
	  <!-- Adding physics coefficients to wheels and caster wheels -->
    <gazebo reference="chefbot_wheel_left_link">
	    <mu1>1.0</mu1>
	    <mu2>1.0</mu2>
	    <kp>1000000.0</kp>
	    <kd>100.0</kd>
	    <minDepth>0.001</minDepth>
	    <maxVel>1.0</maxVel>
	  </gazebo>

	  <gazebo reference="chefbot_wheel_right_link">
	    <mu1>1.0</mu1>
	    <mu2>1.0</mu2>
	    <kp>1000000.0</kp>
	    <kd>100.0</kd>
	    <minDepth>0.001</minDepth>
	    <maxVel>1.0</maxVel>
	  </gazebo>

	  <gazebo reference="chefbot_caster_front_link">
	    <mu1>0.0</mu1>
	    <mu2>0.0</mu2>
	    <kp>1000000.0</kp>
	    <kd>100.0</kd>
	    <minDepth>0.001</minDepth>
	    <maxVel>1.0</maxVel>
	  </gazebo>

	  <gazebo reference="chefbot_caster_back_link">
	    <mu1>0.0</mu1>
	    <mu2>0.0</mu2>
	    <kp>1000000.0</kp>
	    <kd>100.0</kd>
	    <minDepth>0.001</minDepth>
	    <maxVel>1.0</maxVel>
	  </gazebo>

    <!-- set our base as the bumper -->
	  <gazebo reference="base_link">
	    <mu1>0.3</mu1>
	    <mu2>0.3</mu2>
	    <sensor type="contact" name="bumpers">
	      <always_on>1</always_on>
	      <update_rate>50.0</update_rate>
	      <visualize>true</visualize>
	      <contact>
	        <collision>base_footprint_collision_base_link</collision>
	      </contact>
	    </sensor>
	  </gazebo>

    <!-- set cliff sensors, we have three cliff sensors in this case -->
	  <gazebo reference="cliff_sensor_left_link">
	    <sensor type="ray" name="cliff_sensor_left">
	      <always_on>true</always_on>
	      <update_rate>50</update_rate>
	      <visualize>true</visualize>
	      <ray>
	        <scan>
	          <horizontal>
	            <samples>50</samples>
	            <resolution>1.0</resolution>
	            <min_angle>-0.0436</min_angle>  <!-- -2.5 degree -->
	            <max_angle>0.0436</max_angle> <!-- 2.5 degree -->
	          </horizontal>
<!--            Can't use vertical rays until this bug is resolved: -->
<!--            https://bitbucket.org/osrf/gazebo/issue/509/vertical-sensor-doesnt-works -->
<!-- 	          <vertical> -->
<!-- 	            <samples>50</samples> -->
<!-- 	            <resolution>1.0</resolution> -->
<!-- 	            <min_angle>-0.0436</min_angle>  -2.5 degree -->
<!-- 	            <max_angle>0.0436</max_angle> 2.5 degree -->
<!-- 	          </vertical> -->
	        </scan>
	        <range>
	          <min>0.01</min>
	          <max>0.15</max>
	          <resolution>1.0</resolution>
	        </range>
	      </ray>
	    </sensor>
	  </gazebo>

	  <gazebo reference="cliff_sensor_right_link">
	    <sensor type="ray" name="cliff_sensor_right">
	      <always_on>true</always_on>
	      <update_rate>50</update_rate>
	      <visualize>true</visualize>
	      <ray>
	        <scan>
	          <horizontal>
	            <samples>50</samples>
	            <resolution>1.0</resolution>
	            <min_angle>-0.0436</min_angle>  <!-- -2.5 degree -->
	            <max_angle>0.0436</max_angle> <!-- 2.5 degree -->
	          </horizontal>
<!--            Can't use vertical rays until this bug is resolved: -->
<!--            https://bitbucket.org/osrf/gazebo/issue/509/vertical-sensor-doesnt-works -->
<!-- 	          <vertical> -->
<!-- 	            <samples>50</samples> -->
<!-- 	            <resolution>1.0</resolution> -->
<!-- 	            <min_angle>-0.0436</min_angle>  -2.5 degree -->
<!-- 	            <max_angle>0.0436</max_angle> 2.5 degree -->
<!-- 	          </vertical> -->
	        </scan>
	        <range>
	          <min>0.01</min>
	          <max>0.15</max>
	          <resolution>1.0</resolution>
	        </range>
	      </ray>
	    </sensor>
	  </gazebo>

	  <gazebo reference="cliff_sensor_front_link">
	    <sensor type="ray" name="cliff_sensor_front">
	      <always_on>true</always_on>
	      <update_rate>50</update_rate>
	      <visualize>true</visualize>
	      <ray>
	        <scan>
	          <horizontal>
	            <samples>50</samples>
	            <resolution>1.0</resolution>
	            <min_angle>-0.0436</min_angle>  <!-- -2.5 degree -->
	            <max_angle>0.0436</max_angle> <!-- 2.5 degree -->
	          </horizontal>
<!-- 	          Can't use vertical rays until this bug is resolved: -->
<!--            https://bitbucket.org/osrf/gazebo/issue/509/vertical-sensor-doesnt-works -->
<!-- 	          <vertical> -->
<!-- 	            <samples>50</samples> -->
<!-- 	            <resolution>1.0</resolution> -->
<!-- 	            <min_angle>-0.0436</min_angle>  -2.5 degree -->
<!-- 	            <max_angle>0.0436</max_angle> 2.5 degree -->
<!-- 	          </vertical> -->
	        </scan>
	        <range>
	          <min>0.01</min>
	          <max>0.15</max>
	          <resolution>1.0</resolution>
	        </range>
	      </ray>
	    </sensor>
	  </gazebo>

    <!-- adding gyro sensors, the type is going to be imu -->
	  <gazebo reference="gyro_link">
	   <sensor type="imu" name="imu">
        <always_on>true</always_on>
        <update_rate>50</update_rate>
        <visualize>false</visualize>
        <imu>
          <noise>
            <type>gaussian</type>
	          <rate>
	            <mean>0.0</mean>
	            <stddev>${0.0014*0.0014}</stddev> <!-- 0.25 x 0.25 (deg/s) -->
	            <bias_mean>0.0</bias_mean>
	            <bias_stddev>0.0</bias_stddev>
	          </rate>
		        <accel> <!-- not used in the plugin and real robot, hence using tutorial values -->
			        <mean>0.0</mean>
			        <stddev>1.7e-2</stddev>
			        <bias_mean>0.1</bias_mean>
			        <bias_stddev>0.001</bias_stddev>
		        </accel>
          </noise>
	      </imu>
      </sensor>
	  </gazebo>

    <!--
      kobuki plugin takes in three cliff sensors, imu, and bumper data
      In addition, this plugin will need to know wheel_separation and wheel_diameter
     -->
	  <gazebo>
	    <plugin name="kobuki_controller" filename="libgazebo_ros_kobuki.so">
	      <publish_tf>1</publish_tf>
	      <left_wheel_joint_name>wheel_left_joint</left_wheel_joint_name>
	      <right_wheel_joint_name>wheel_right_joint</right_wheel_joint_name>
	      <wheel_separation>.30</wheel_separation>
	      <wheel_diameter>${WHEEL_RADIUS*2}</wheel_diameter>
	      <torque>18.0</torque>
	      <velocity_command_timeout>0.6</velocity_command_timeout>
	      <cliff_sensor_left_name>cliff_sensor_left</cliff_sensor_left_name>
	      <cliff_sensor_center_name>cliff_sensor_front</cliff_sensor_center_name>
	      <cliff_sensor_right_name>cliff_sensor_right</cliff_sensor_right_name>
	      <cliff_detection_threshold>0.04</cliff_detection_threshold>
	      <bumper_name>bumpers</bumper_name>
        <imu_name>imu</imu_name>
	    </plugin>
	  </gazebo>
  </xacro:macro>
</robot>
```

Now we want to call **kobuki_sim** macro to bring in properties of sensors and simulate in gazebo
```xml
<!-- myturtle_base.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  ...
  <!-- Macro defined in myturtle_base_gazebo.urdf.xacro -->
  <kobuki_sim/>
</robot>
```
When you launch the myturtle, you shuld be able to see some [INFO] about kobuki data
<br></br>
![Kobuki Controller](images/kobuki_controller.png)

#### Implement Kinect
We add a file under /myturtle_description/meshes/urdf/sensors/kinect.urdf.xacro
```xml
~/catkin_ws/src/myturtle
  /myturtle_gazebo
    /launch
      /include
        myturtle_base.launch.xml
      myturtle_empty_world.launch
    CMakeList.txt
    package.xml
  /myturtle_description
    /meshes
    /robots
      myturtle_circles_kinect.xacro
    /urdf
      /sensors
        kinect.urdf.xacro
      myturtle_base.urdf.xacro
      myturtle_base_gazebo.urdf.xacro
      myturtle_library.urdf.xacro
      myturtle_properties.urdf.xacro
    CMakeList.txt
    package.xml
```
##### Include Kinect
```xml
<!-- myturtle_library.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  ...
  <!-- Kinect Sensor -->
  <xacro:include filename="$(find myturtle_description)/urdf/sensors/kinect.urdf.xacro" />
</robot>
```
##### Define RGB Camera
Let's defined some values to position the kinect camera. These values are for turtlebot model
```xml
<!-- myturtle_properties.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  <property name="cam_px" value="-0.087" />
	<property name="cam_py" value="-0.0125" />
	<property name="cam_pz" value="0.2870" />
	<property name="cam_or" value="0" />
	<property name="cam_op" value="0" />
	<property name="cam_oy" value="0" />
</robot>
```
Setting up joint and link for rgb camera
```xml
<!-- kinect.urdf.xacro -->
<?xml version="1.0"?>
<robot name="sensor_kinect" xmlns:xacro="http://ros.org/wiki/xacro">
  <xacro:macro name="sensor_kinect" param="parent">
    <!-- include camera position values from myturtle properties -->
    <xacro:include filename="$(find myturtle_description)/urdf/myturtle_properties.urdf.xacro" />

    <joint name="camera_rgb_joint" type="fixed">
      <origin xyz="${cam_px} ${cam_py} ${cam_pz}" rpy="${cam_or} ${cam_op} ${cam_oy}"/>
      <parent link="${parent}"/>
      <child link="camera_rgb_frame" />
    </joint>
    <link name="camera_rgb_frame">
      <inertial>
        <mass value="0.001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
    </link>
    <joint name="camera_rgb_optical_joint" type="fixed">
      <origin xyz="0 0 0" rpy="${-M_PI/2} 0 ${-M_PI/2}" />
      <parent link="camera_rgb_frame" />
      <child link="camera_rgb_optical_frame" />
    </joint>
    <link name="camera_rgb_optical_frame">
      <inertial>
        <mass value="0.001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
    </link>
  </xacro:macro>
</robot>
```
##### Define Kinect Camera Link
```xml
<!-- kinect.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  <xacro:macro>
      ...
      <joint name="camera_joint" type="fixed">
        <origin xyz="-0.031 ${-cam_py} -0.016" rpy="0 0 0"/>
        <parent link="camera_rgb_frame"/>
        <child link="camera_link"/>
      </joint>
      <link name="camera_link">
        <visual>
         <origin xyz="0 0 0" rpy="0 0 ${M_PI/2}"/>
          <geometry>
           <!-- load kinect 3d model so that we can visualize where the robot is -->
           <mesh filename="package://turtlebot_description/meshes/sensors/kinect.dae"/>
          </geometry>
        </visual>
    	  <collision>
          <origin xyz="0.0 0.0 0.0" rpy="0 0 0"/>
    	    <geometry>
    	      <box size="0.07271 0.27794 0.073"/>
    	    </geometry>
    	  </collision>
        <inertial>
          <mass value="0.001" />
          <origin xyz="0 0 0" />
          <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                   iyy="0.0001" iyz="0.0"
                   izz="0.0001" />
        </inertial>
      </link>
  </xacro:macro>
</robot>
```

##### Define Depth Camera
The fixed joints & links below are usually published by static_transformers launched by the OpenNi launch files.
However, for Gazebo simulation we need them, so we add them after the camera link.
```xml
<?xml version="1.0"?>
<robot>
  <xacro:macro>
    ...
    <joint name="camera_depth_joint" type="fixed">
  	  <origin xyz="0 ${2 * -cam_py} 0" rpy="0 0 0" />
  	  <parent link="camera_rgb_frame" />
  	  <child link="camera_depth_frame" />
  	</joint>
  	<link name="camera_depth_frame">
      <inertial>
        <mass value="0.001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
  	</link>
  	<joint name="camera_depth_optical_joint" type="fixed">
  	  <origin xyz="0 0 0" rpy="${-M_PI/2} 0 ${-M_PI/2}" />
  	  <parent link="camera_depth_frame" />
  	  <child link="camera_depth_optical_frame" />
  	</joint>
  	<link name="camera_depth_optical_frame">
      <inertial>
        <mass value="0.001" />
        <origin xyz="0 0 0" />
        <inertia ixx="0.0001" ixy="0.0" ixz="0.0"
                 iyy="0.0001" iyz="0.0"
                 izz="0.0001" />
      </inertial>
  	</link>
  </xacro:macro>
</robot>
```

##### Call Kinect Macro
We need to actually use the Macro by using the macro's name as a tag
```xml
<?xml version="1.0"?>
<robot name="turtlebot" xmlns:xacro="http://ros.org/wiki/xacro">
  ...
  <kobuki/>
  <!-- We declared the "parent" as a parameter to define the parent link for joint -->
  <sensor_kinect parent="base_link" />
</robot>
```
We should be able to see a kinect mounted on the robot now
<br></br>
![Kinect On MyTurtle](images/myturtle_kinect.png)

##### Setup Gazebo Properties for Camera
```xml
<!-- myturtle_gazebo.urdf.xacro -->
<?xml version="1.0"?>
<robot name="turtlebot_gazebo" xmlns:xacro="http://ros.org/wiki/xacro">
  <!-- Microsoft Kinect / ASUS Xtion PRO Live for simulation -->
  <xacro:macro name="turtlebot_sim_3dsensor">
    <gazebo reference="camera_link">
      <sensor type="depth" name="camera">
        <always_on>true</always_on>
        <update_rate>20.0</update_rate>
        <camera>
          <horizontal_fov>${60.0*M_PI/180.0}</horizontal_fov>
          <image>
            <format>R8G8B8</format>
            <width>640</width>
            <height>480</height>
          </image>
          <clip>
            <near>0.05</near>
            <far>8.0</far>
          </clip>
        </camera>
        <plugin name="kinect_camera_controller" filename="libgazebo_ros_openni_kinect.so">
          <cameraName>camera</cameraName>
          <alwaysOn>true</alwaysOn>
          <updateRate>10</updateRate>
          <imageTopicName>rgb/image_raw</imageTopicName>
          <depthImageTopicName>depth/image_raw</depthImageTopicName>
          <pointCloudTopicName>depth/points</pointCloudTopicName>
          <cameraInfoTopicName>rgb/camera_info</cameraInfoTopicName>
          <depthImageCameraInfoTopicName>depth/camera_info</depthImageCameraInfoTopicName>
          <frameName>camera_depth_optical_frame</frameName>
          <baseline>0.1</baseline>
          <distortion_k1>0.0</distortion_k1>
          <distortion_k2>0.0</distortion_k2>
          <distortion_k3>0.0</distortion_k3>
          <distortion_t1>0.0</distortion_t1>
          <distortion_t2>0.0</distortion_t2>
          <pointCloudCutoff>0.4</pointCloudCutoff>
        </plugin>
      </sensor>
    </gazebo>
  </xacro:macro>
</robot>
```

##### Call Kinect Simulator Macro
We need to call the gazebo properties for kinect by calling the macro after all the joints and links declaration of kinect.urdf.xacro
```xml
<!-- kinect.urdf.xacro -->
<?xml version="1.0"?>
<robot>
  <xacro:macro>
    ...
    <turtlebot_sim_3dsensor/>
  </xacro:macro>
</robot>
```

#### Velocity Muxer
```xml
<!-- myturtle_base.launch.xml -->
<launch>
  ...
  <node />
  <!-- Velocity muxer -->
  <node pkg="nodelet" type="nodelet" name="mobile_base_nodelet_manager" args="manager"/>
  <node pkg="nodelet" type="nodelet" name="cmd_vel_mux"
        args="load yocs_cmd_vel_mux/CmdVelMuxNodelet mobile_base_nodelet_manager">
    <param name="yaml_cfg_file" value="$(find turtlebot_bringup)/param/mux.yaml" />
    <remap from="cmd_vel_mux/output" to="mobile_base/commands/velocity"/>
  </node>
</launch>
```

#### Robot State Publisher
```xml
<!-- myturtle_empty_world.launch -->
<launch>
  ...
  <include> ... </include>

  <!-- robot state publisher -->
  <node pkg="robot_state_publisher" type="robot_state_publisher" name="robot_state_publisher">
    <param name="publish_frequency" type="double" value="30.0" />
  </node>
</launch>
```
#### Setting Up Fake Laser
We use kinect to fake a laser by using depthimage_to_laserscan package
```xml
<launch>
  ...
  <node> ... </node>

  <!-- Fake laser -->
  <node pkg="nodelet" type="nodelet" name="laserscan_nodelet_manager" args="manager"/>
  <node pkg="nodelet" type="nodelet" name="depthimage_to_laserscan"
        args="load depthimage_to_laserscan/DepthImageToLaserScanNodelet laserscan_nodelet_manager">
    <param name="scan_height" value="10"/>
    <param name="output_frame_id" value="/camera_depth_frame"/>
    <param name="range_min" value="0.45"/>
    <remap from="image" to="/camera/depth/image_raw"/>
    <remap from="scan" to="/scan"/>
  </node>
</launch>
```
