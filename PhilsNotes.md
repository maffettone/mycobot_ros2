## Need to set on RPi shell

```bash
ros2-init # Sets up the environment for ROS2 (builtin to bashrc, same as the ROS2 Icon.)
export ROS_DOMAIN_ID=1 # Best to be explicit.
```


## Multicast - How ROS Nodes Communicate
Multicast ran into plenty of challenges, especially with dev containers. The best solution I found was a link-local network directly connecting the host machine and the RPi.
To accomplish this, connect the ethernet port of the RPi to the host machine. 
Confirm the interface name of the ethernet port on both machines using the command:

```bash
ip addr
```

If it is for example `eth0` on the RPi and `eno1` on the host machine, you can set up a link-local network by running the following commands.
On the host machine:

```bash
sudo ip addr add 169.254.100.1/16 dev eno1
sudo ip link set eno0 up
```

This adds an IP address to the ethernet port on the host machine and brings it up. The IP address is a link-local address, which means it is only valid on the local network segment. The `/16` subnet mask is standard for link-local addresses, anf the `169.254.x.x` range is reserved for link-local addresses. The `eno1` interface name may be different on your machine, so make sure to check the output of `ip addr` to find the correct name.

The second command brings the ethernet port up, in case it was down.

On the RPi:

```bash
sudo ip addr add 169.254.100.2/16 dev eth0
sudo ip link set eth0 up
sudo ip route add default dev eth0
```

These commands are similar to the above, with a new, unique IP address for the RPi. The final command adds a default route for traffic to the eth0 device. The default route is only necessary for the RPi because it does not have a default route set up, and multicast needs to know where to go.
The host machine should already have a default route set up. `ip route` will show the routing table.


## Use the same version of the repository on both machines

The syntax and paths for the packages will have changed since it was compiled and shipped on the RPi.
You can copy these over from the host machine to the RPi using `rsync` or `scp`. This will also test your network setup (in part).

```bash
# ssh and backup the original source using IP from above
# Username is `er` and password is `Elephant` by default.
ssh er@169.254.100.2
mv ~/colcon_ws/src/mycobot_ros2 ~/original_source_backup
exit
# Copy the source from the host machine to the RPi.
# From inside the dev container:
scp -r $(pwd) er@169.254.100.2:~/colcon_ws/src/
# Now head back into the RPi and build the package (eta 2 min).
ssh er@169.254.100.2
cd ~/colcon_ws
ros2-init
colcon build
```


## Network Sanity Checks

You can check the network connection using some demo nodes. 
On the RPi:

```bash
ros2-init
export ROS_DOMAIN_ID=1
ros2 run demo_nodes_cpp talker
```

On the dev container:

```bash
ros2 run demo_nodes_cpp listener
```

You should see the listener node receiving messages from the talker node. If you don't, check your network setup and make sure the IP addresses are correct.

### Check the following

- `ip addr` on both machines should show interfaces on the 169.254.100.x network.
- `echo $ROS_DOMAIN_ID` on both machines should be 1.
- `ip route` on the RPi should show a default route to eth0 at the top: `default dev eth0 scope link`


## URDF, SRDF, MoveIt

- The mycobot descriptions and urdf describe the links as "joint1" etc and the joints as "joint2_to_joint1" etc. Not how I would have done it, but I stay consistent with the package.
- It was a helpful exercise to create the SRDF and MoveIt configuration files from scratch. This was necessary as the moveit wizard does not work with ROS2 Galactic which is what the RPi is running. (Outdated).

## Missing Packages on RPi
- `ros-galactic-ros2-control`
- `chrony`


## Clock sync
The RPi will not have a clock sync with the host machine, so you will need to set the clock manually. This can be done using the `date` command from your host machine. You will  need to set up ssh keys to minimize latency.:

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id er@169.254.100.2
ssh er@169.254.100.2 "sudo date -s @$(date +%s.%3N)"
```

## Clock sync with chrony
You can also use `chrony` to keep the clocks in sync. This is a more robust solution, but requires some setup.
Edit the `/etc/chrony/chrony.conf` on the local machine and add the following lines:

```bash
allow 169.254.100.2
```
Then restart the chrony service:

```bash
sudo systemctl restart chronyd
sudo chronyc tracking
```

and check that its listening

```bash
sudo chronyc sources
```

Then on the Raspberry Pi client ( making sure chorony is installed), add the following to `/etc/chrony/chrony.conf`:

```bash
server 169.254.100.1 iburst prefer
```

Restart the chrony service on the RPi:

```bash
sudo systemctl restart chronyd
chronyc tracking
chronyc sources
```

Enable chrony to start on boot on the RPi:

```bash
sudo systemctl enable chrony
```

## Using MoveIt, Rviz & other GUIs
By now, you should have a functioning dev container opened and ready to use. Be sure to run sanity checks before moving on.

### Simple_GUI for RPi
Launch simple_gui, this will only be able to run on the RPi

```bash
ros2 launch mycobot_280pi simple_gui.launch.py gui:='true' rviz:='false'
```
*Note: rviz may open even though it is set to false, close it anyways (most likely will crash RPi if left open too long)*

You should see a simple gui in Chinese. Change angle values and hit the button below to move the robot.

### Pub_Sub_driver GUI for Host machine and RPi
This will allow you to control the robot from your host machine. You should see another simple GUI (this time in English). Changing the angles of joints or the status of the gripper should be made in Rviz and on the actual robot. *Rviz is not currently configured to display the gripper.```

The subscriber will listen for instructions from the publsiher, similar to the talker and listener demo shown earlier.

Run the subscriber driver on the RPi:

```bash
ros2 launch mycobot_280pi pub_sub_driver_gui.launch.py driver:='true' rviz:='false' gui:='false'
```

Run the publisher driver on the host machine:
```bash
ros2 launch mycobot_280pi pub_sub_driver_gui.launch.py driver:='false' rviz:='true' gui:='true'
```

### MoveIt

This will allow you to move the robot in Rviz, create a plan, and execute it.

On the RPi, run the action_driver:
```bash
ros2 run mycobot_280pi action_driver
```

On your host machine, run MoveIt:
```bash
ros2 launch mycobot_moveit_config moveit.launch.py
```
Drag the ball at the end of the robotic arm to move it to your desired position. Press the plan button to see a plan on how it plans to get from it's intial point to it's desired point. Then press 'Plan and Execute' to see it move.