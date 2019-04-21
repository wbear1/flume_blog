# Sink设计与实现

Flume Sink的工作流程与Source类似：
![flow](https://github.com/wbear1/flume_blog/blob/master/img/sink/flow.png)

关于sinkProcessor的介绍如下，参考自http://stackoverflow.com/questions/42547921/what-is-the-advantage-of-having-several-sinks-grouped-in-sink-groups-in-flume

The Flume configuration framework instantiates one sink runner per sink group to run a sink group. Each sink group can contain an arbitrary number of sinks. The sink runner continuously asks the sink group to ask one of its sinks to read events from its own channel. Sink groups are typically used to send data between tiers in either a load-balancing or failover fashion.

It is important to understand that all sinks within a sink group are not active at the same time; only one of them is sending data at any point in time. Therefore, sink groups should not be used to clear off the channel faster—in this case, multiple sinks should simply be set to operate by themselves with no sink group, and they should be configured to read from the same channel.

Sink groups are defined in the following way:

agent.sinkgroups = sg1 sg2
Each sink group is then configured with a set of sinks that are part of the group. The list of sinks in the active set of sinks takes precedence over the lists of sinks specified as part of sink groups.

agent.sinks = s1 s2 s3 s4
agent.sinkgroups.sg1.sinks = s1 s2
agent.sinkgroups.sg2.sinks = s3 s4
Each sink in a sink group has to be configured separately. This includes configuration with regard to which channel the sink reads from, which hosts or clusters it writes data to, etc. Ideally, if several sinks are set up in a sink group, all the sinks will read from the same channel—this helps clear data in the current tier at a reasonable pace, yet ensure the data is being sent to multiple machines in a way that supports load balancing and failover.

For cases where it is important to clear the channel faster than a single sink group is able to do, but it is also required that the agent be set up to send data to multiple hosts, multiple sink groups can be added, each with sinks that have similar configuration. This ensures that more connections are open per agent to a destination agent, while also allowing data to be pushed to more than one agent if required. This allows the channel to be cleared faster, while making sure load balancing and failovers happen automatically.

Sink processors are the components that decide which sink is active at any point in time. Note that sink processors are different from sink runners. The sink runner actually runs the sink, while the sink processor decides which sink should pull events from its channel. When the sink runner asks the sink group to tell one of its sinks to pull events out of its channel and write them to the next hop (or to storage), the sink processor is the component that actually selects the sink that does this processing.

Flume comes bundled with two sink processors: the load-balancing sink processor and the failover sink processor.

**Load-Balancing Sink Processor**

A sink group with a load-balancing sink processor, which will select one among all the sinks in the sink group to process events from the channel. The order of selection of sinks can be configured to be random or round-robin. If the order is set to random, one among the sinks in the sink group is selected at random to remove events from its own channel and write them out. The round-robin fashion: each process loop calls the process method of the next sink in the order in which they are specified in the sink group definition. If that sink is writing to a failed agent or to an agent that is too slow, causing timeouts, the sink processor will select another sink to write data. The load-balancing sink processor is configured in the following way:

This configuration sets the sink group to use a load-balancing sink processor that selects one of s1, s2, s3, or s4 at random. If one of the sinks (or more accurately, the agent that the sink is sending data to) fails, the sink will be blacklisted with the backoff period starting at 250 milliseconds and then increasing exponentially until it reaches 10 seconds. After this point, the sink backs off for 10 seconds each time a write fails, until it is able to write data successfully, at which point the backoff is reset to 0. If the value of the selector parameter is set to round_robin, s1 is asked to process data first, followed by s2, then s3, then s4, and s1 again.

agent.sinks = s1 s2 s3 s4
agent.sinkgroups = sg1
agent.sinkgroups.sg1.sinks = s1 s2 s3 s4
agent.sinkgroups.sg1.processor.type = load_balance
agent.sinkgroups.sg1.processor.selector = random
agent.sinkgroups.sg1.processor.backoff = true
agent.sinkgroups.sg1.processor.selector.maxTimeOut = 10000
This configuration means that only one sink is writing data from each agent at any point in time. This can be fixed by adding multiple sink groups with load-balancing sink processors with similar configuration.

**Failover Sink Processor**

The failover sink processor selects a sink from the sink group based on priority. The sink with the highest priority writes data until it fails, and then the sink with the highest priority among the other sinks in the group is picked. A different sink is selected to write the data only when the current sink writing the data fails. This means even though it is possible that the agent with highst priority may have failed and come back online, the failover sink processor does not make the sink writing to that agent active until the currently active sink hits an error. This ensures that all agents on the second tier have one sink from each machine writing to them when there is no failure, and only on failure will certain machines see more incoming data.

If no priority is set for a specific sink, the priority of the sink is determined based on the order of the sinks specified in the sink group configuration. Each time a sink fails to write data, the sink is considered to have failed and is blacklisted for a brief period of time. This blacklist time interval (similar to the backoff period in the load-balancing sink processor) increases with each consecutive attempt that results in failure, until the value specified by maxpenalty is reached (in milliseconds). Once the blacklist interval reaches this value, further failures will result in the sink being tried after that many milliseconds. Once the sink successfully writes data after this, the backoff period is reset to 0. Take a look at the following example:

agent.sinks = s1 s2 s3 s4
agent.sinkgroups.sg1.sinks = s1 s2 s3 s4
agent.sinkgroups.sg1.processor.type = failover
agent.sinkgroups.sg1.processor.priority.s2 = 100
agent.sinkgroups.sg1.processor.priority.s1 = 90
agent.sinkgroups.sg1.processor.priority.s4 = 110
agent.sinkgroups.sg1.processor.maxpenalty = 10000