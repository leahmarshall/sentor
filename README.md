﻿


> # ROS messages monitoring node

Continuously monitor topic messages. Send warnings and execute other processes when certain conditions on the messages are satisfied. 

## Launch

Example launch command:

```sh
roslaunch sentor sentor.launch config_file:=$(rospack find sentor)/config/execute.yaml
```

## Config

The config file contains the list of topics to be monitored and the definition, for each, of when we want to be alerted. It can also contain, for each monitored topic, an optional list of processes to be executed after sending the alert.

```yaml
- name : '/row_detector/path_error'
  signal_when : 'not published'
  signal_lambdas :
    - "lambda msg : math.isnan(msg.y)"
  timeout : 0.0
  

- name: "/topological_navigation/feedback"
  signal_lambdas:
  - "lambda msg: msg.feedback.route == 'WayPoint1'"
  execute:
  - action:
      user_msg: "Navigating to charging node."
      namespace: "/topological_navigation"
      package: "topological_navigation"
      action_spec: "GotoNodeAction"
      goal_args:
      -  "goal.target = 'WayPoint45'"
  - shell:
      user_msg: "Cow says 'moo'."
      cmd_args:
      - "cowsay"
      - "moo"
  lock_exec: False
  timeout: 0.0                 


- name: "/topological_navigation/feedback"
  signal_lambdas:
  - "lambda msg: msg.feedback.route == 'WayPoint45'"
  execute:
  - call:
      user_msg: "Teleporting the robot."
      service_name: "/gazebo/set_model_state"
      service_args:
      -  "req.model_state.model_name = 'thorvald_ii'"
      -  "req.model_state.pose.orientation.w = 1.0"
  - sleep:
      user_msg: "zzz"
      duration: 3.0
  - publish:
      user_msg: "Relocalising the robot."
      topic_name: "/initialpose"
      topic_latched: False
      topic_args:
      -  "msg.header.frame_id = '/map'"
      -  "msg.pose.pose.orientation.w = 1.0"
  lock_exec: False
  timeout: 0.0
```
Top-level arguments:
- `name`: is the name of the topic to monitor
- `signal_when`: optional, can be either 'not published' or 'published'. Respectively, it will send a warning when the topic is not published or when it is.
- `signal_lambdas`: optional, it's a list of (pythonic) lambda expressions such that when they are satisfied a warning is sent. You can use the python package `math` in your lambda expressions.
- `execute`: optional, a list of processes to execute after the warning is sent. These will be executed in sequence. See 'Child arguments of `execute`' below.
- `lock_exec`: optional (default=False), lock out other threads while this one is executing its sequence of processes.
- `timeout`: optional (default=0.1), amount of time (in seconds) for which the signal has to be satisfied before sending the warning.

Child arguments of `execute`:
- `call`: optional, call a rosservice.
- `publish`: optional, publish to a rostopic.
- `action`: optional, send a goal for an actionlib action.
- `sleep`: optional, put the sentor node to sleep.
- `shell`: optional, execute a shell command.  

Child arguments of `call`:
- `user_msg`: optional, publish your own (string) message to the topic `/sentor/event` when `call` is executed.
- `service_name`: the name of the service you are calling.
- `service_args`: a list of service arguments specified in the service request class. Each arg must be prefixed by `req.`

Child arguments of `publish`:
- `user_msg`: optional, publish your own (string) message to the topic `/sentor/event` when `publish` is executed.
- `topic_name`: the name of the topic you are publishing to. 
- `topic_latched`: boolean specifying whether you are latching the topic (or not).
- `topic_args`: a list of topic arguments specified in the topic's message class. Each arg must be prefixed by `msg.`

Child arguments of `action`:
- `user_msg`: optional, publish your own (string) message to the topic `/sentor/event` when `action` is executed.
- `namespace`: the namespace of the action.
- `package`: the ros package from which the action specification is retrieved. Specifically the action specification is retrieved from `package.msg`. 
- `action_spec`: the action specification.
- `goal_args`: a list of goal arguments specified in the action spec's goal class. Each arg must be prefixed by `goal.`

Child arguments of `sleep`:
- `user_msg`:  optional, publish your own (string) message to the topic `/sentor/event` when `sleep` is executed.
- `duration`: sleep the sentor node for `duration` seconds.

Child arguments of `shell`:
- `user_msg`: optional, publish your own (string) message to the topic `/sentor/event` when `shell` is executed.
- `cmd_args`: a list of shell command components.

## Using sentor with this example config
You will need the RASberry repo (<a href="https://github.com/LCAS/RASberry">get it here</a>) and all its dependencies. Create a file `.rasberryrc` in your home directory and put the following inside it:

`export ROBOT_NAME="thorvald_023"`<br />
`export SCENARIO_NAME="sim_riseholme-uv_poly_act"`  

Then issue this command:
```sh
cd $(rospack find rasberry_bringup)/tmule && tmule -c rasberry-simple_robot_corner_hokuyos.yaml -W 3 launch
```
A Thorvald robot will appear in a gazebo environment and a topological map will be displayed in rviz. Then launch sentor as per the example launch command given above. Then send the robot to the `WayPoint1` node in the topological map. 

You should view gazebo, rviz, the terminal in which you have launched the sentor node and another terminal in which you have echoed the topic `/sentor/event`.
