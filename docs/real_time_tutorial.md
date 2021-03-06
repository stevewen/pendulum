# Instructions to test real-time capabilities

## Requirements

### RTOS

To test the demo real-time capabilities to use a RTOS is strongly recommended but not mandatory. For Linux users you can follow [these instructions](real_time_linux.md).

### TLSF allocator

ROS 2 offers support for the TLSF (Two Level Segregate Fit) allocator, which was designed to meet real-time requirements:

* https://github.com/ros2/realtime_support/tree/master/tlsf_cpp
* https://index.ros.org/doc/ros2/Tutorials/Allocator-Template-Tutorial/

In order to use the TLSF allocator for the pendulum demo the package realtime_support must be compiled in the workspace. If it's not found we would get an error when enabling the TLSF allocator.

### DDS implementation configuration
First of all make sure you are running the demo in a RTOS.
ROS 2 uses DDS as communication middleware. Several DDS implementations are currently supported by ROS 2. In order to achieve real-time capabilities the DDS also has to be configured properly. Each DDS has it's own configuration method, usually based in a xml QoS profile file. Depending on the DDS implementation it is possible to configure the middleware threads priority, lock memory, use static memory allocation, etc.  

We will extend this section in the future providing detailed instructions about how to configure each DDS implementation. (Help is welcome!)

## Examples

For the moment it is not possible in this demo to configure the real-time settings using the ROS 2 launch system. For this reason if we want to tune the real-time settings we have to launch the demo executables using ros2 run.

### Set real-time priority

We can change the process priority and set a real-time FIFO priority. This will set the priority of the thread where the ROS 2 executor is running.

```
ros2 run pendulum_demo pendulum_demo --priority 80
```

Make sure, we have permissions to change the priority of process in the OS.

### Set CPU affinity

To bind the application to a specific CPU core we can use the option `--cpu-affinity` to set the CPU mask. For example if we want to bind the process to CPU 3 (or 2 starting from 0) we set the CPU mask to 4 (100 in binary).

```
ros2 run pendulum_demo pendulum_demo --priority 80 --cpu-affinity 4
```

### Lock memory

If our process generates memory page faults when running a real-time task we would suffer an undetermined block time. To prevent this, we must pre-allocate all the dynamic memory used by the process and lock it into RAM.

We can enable memory locking with the option `--lock-memory`.The command will pre-fault memory until no more page faults are seen. This usually allocated a high amount of memory (~8GB) so make sure there is enough memory in the system.

```
ros2 run pendulum_demo pendulum_demo --lock-memory
```

Additionally, we can specify the amount of memory we want to pre-allocate with the option `--lock-memory-size`. For example we can pre-allocate 50 MB

```
ros2 run pendulum_demo pendulum_demo --lock-memory-size 50000
```

If we get the following error message this means we don't have permissions or that we can't allocate the requested amount of memory.

```
mlockall failed: Cannot allocate memory
Couldn't lock  virtual memory.
```

### Set topic deadline QoS

For aplications requiring more control over data delivered [new DDS QoS were introduced](https://design.ros2.org/articles/qos_deadline_liveliness_lifespan.html). One of this QoS is the deadline policy.

>The deadline policy establishes a contract for the amount of time allowed between messages. For Subscriptions it establishes the maximum amount of time allowed to pass between receiving messages. For Publishers it establishes the maximum amount of time allowed to pass between sending messages

In this demo, the topics between the controller and driver nodes use by default this QoS. We can configure the deadlines for publishers and subscriptions with the option `--deadline` and specifying the required deadlines in milliseconds.  

```
ros2 run pendulum_demo pendulum_demo --deadline 2
```

It is important to take into account when setting this value if intra-process communications are used or not. Latencies for intra-process communications are significantly lower than when the networking stack is used.

### TLSF allocator

It is posible to specify a custom allocator to be used by the executor. In this demo the TLSF allocator can be used with the option
`--use-tlsf`. This allocator provides bounded allocation times during runtime.

### Launch standalone nodes

In the previous examples we launched a process containing both the controller and simulation/driver nodes. This means that a single executor will spin both nodes. The user may want to analyze simpler scenarios. For this reason, it is possible to launch each node in a separate process. For example we may want to launch the simulation without real-time settings and experiment just with the controller node. The same real-time options are available for each executable.

To launch each node separately we run:

In terminal 1:
```
ros2 run pendulum_demo pendulum_controller_standalone
```

In terminal 2:
```
ros2 run pendulum_demo pendulum_driver_standalone
```
