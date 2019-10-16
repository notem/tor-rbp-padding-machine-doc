# Tor RBP Prototype Machines

This documentation is an overview of the Random Burst Padding (RBP) Tor padding machine prototypes. 
The padding technique implemented here is the same as used in [Understanding Feature Discovery in Website Fingerprinting Attacks](https://www.researchgate.net/publication/329743510_UNDERSTANDING_FEATURE_DISCOVERY_IN_WEBSITE_FINGERPRINTING_ATTACKS).
The code is found on my [circuit_padding_rbp_machine](https://github.com/notem/tor/tree/circuit_padding_rbp_machine) branch.

NOTE: The (no-delay) Random Break Bursts padding machine is currently non-functional (see *Issues*).

## Design

Random Burst Padding is a padding defense technique that aims to reduce the classification accuracy of website fingerprinting attacks by randomly injecting bursts of fake cells into the traffic flow. These bursts can be injected with the flow of the current burst---Random Extending Bursts (REB)---or against the flow of the current burst---Random Break Bursts (RBB). 

NOTE: In the original work used this technique in combination of the sensitivity mapping technique GradCAM to target important regions of a trace, and found that RBB with packet delays to insure clean burst breaking produced reasonably effective results. Neither of these mechanisms can be implemented in the Tor circuit padding framework, so only the base padding machism is implemented.

Implementing this padding mechanism inside the adaptive padding-style framework is surprisingly straightforward.
This padding machine uses three states and IAT histograms containing only a 0 and infinity bin. 

<pre>
                                                               <-----^
                                                               |     |
+-------------+            +-------------+         +-------------+   |
|             | NONPAD RCV |             |         |             |--->
|             |    or      |             | PAD SND |             | PAD SND
|   STATE 1   | NONPAD SND |   STATE 2   |         |   STATE 3   |
|             +----------->|             +-------->|             |
|             |            |             |         |             |
+-------------+            +------+------+         +------+------+
       ^                 INFINITY |              INFINITY |
       |                 SAMPLED  |              SAMPLED  |
       +--------------------------+-----------------------+
</pre>

The event that causes transitions from STATE 1 to STATE 2 can be either a NONPADDING PACKET RECIEVED event to produce no-delay RBB padding or a NONPADDING PACKET SENT event to produce REB padding.

The purpose of STATE 2 is to probabilistically decide when to start a fake burst by sampling either a zero or infinity delay from its histogram. Similarly, STATE 3 samples from different two-bin histogram to determine when a fake-burst should end. The ratio of the two bin sizes determines the likelyhood that a fake burst will start for every STATE 1 trigger or end for every STATE 3 transition.

For full details, see the [code and docstrings in the source files](https://github.com/notem/tor/blob/circuit_padding_rbp_machine/src/core/or/circuitpadding_machines.c#L457).

## Issues

Since this padding machine heavily relies on 0usec delays to immediantly trigger burst padding, this machine is affected by [issue #31653](https://trac.torproject.org/projects/tor/ticket/31653). The bandaid solution suggested by pulls is used so that the REB machine can function for the time being. 

Strangely, the RBB machine continues to fail despite this solution. The circuit on the relay-side continues to crash with the following error:

<pre>
Oct 15 14:41:40.000 [info] circpad_setup_machine_on_circ(): Registering machine relay_wf_rbb to non-origin circ (1)
Oct 15 14:41:40.000 [info] circpad_machine_spec_transition(): Circuit 0 circpad machine 0 transitioning from 0 to 1
Oct 15 14:41:40.000 [info] circpad_setup_machine_on_circ(): Registering machine relay_wf_rbb to non-origin circ (1)
Oct 15 14:41:40.000 [info] circpad_machine_spec_transition(): Circuit 0 circpad machine 0 transitioning from 0 to 1

============================================================ T= 1571164900
Tor 0.4.1.5 (git-0c4939b383926a6f) died: Caught signal 8
/usr/bin/tor(+0x212919)[0x55a1ec9aa919]
/usr/bin/tor(circpad_machine_schedule_padding+0xf7)[0x55a1ec82b6e7]
/usr/bin/tor(circpad_machine_schedule_padding+0xf7)[0x55a1ec82b6e7]
/usr/bin/tor(circpad_machine_spec_transition+0xe5)[0x55a1ec82afd5]
/usr/bin/tor(circpad_cell_event_nonpadding_received+0x7a)[0x55a1ec82b2da]
/usr/bin/tor(circpad_handle_padding_negotiate+0x160)[0x55a1ec82ce10]
/usr/bin/tor(+0xbb496)[0x55a1ec853496]
/usr/bin/tor(circuit_receive_relay_cell+0x490)[0x55a1ec855b80]
/usr/bin/tor(command_process_cell+0x3f8)[0x55a1ec837958]
/usr/bin/tor(channel_tls_handle_cell+0x39b)[0x55a1ec816c8b]
/usr/bin/tor(+0xa84dc)[0x55a1ec8404dc]
/usr/bin/tor(connection_handle_read+0xa0d)[0x55a1ec8040fd]
/usr/bin/tor(+0x7134e)[0x55a1ec80934e]
/usr/lib/x86_64-linux-gnu/libevent-2.1.so.6(+0x1e8f8)[0x7fb52a4ad8f8]
/usr/lib/x86_64-linux-gnu/libevent-2.1.so.6(event_base_loop+0x53f)[0x7fb52a4ae33f]
/usr/bin/tor(do_main_loop+0xfd)[0x55a1ec80a65d]
/usr/bin/tor(tor_run_main+0x126d)[0x55a1ec7f815d]
/usr/bin/tor(tor_main+0x3a)[0x55a1ec7f54da]
/usr/bin/tor(main+0x19)[0x55a1ec7f5069]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7)[0x7fb528ddcb97]
/usr/bin/tor(_start+0x2a)[0x55a1ec7f50ba]
============================================================ T= 1571164900
Tor 0.4.1.5 (git-0c4939b383926a6f) died: Caught signal 8
/usr/bin/tor(+0x212919)[0x55a1ec9aa919]
/usr/bin/tor(circpad_machine_schedule_padding+0xf7)[0x55a1ec82b6e7]
/usr/bin/tor(circpad_machine_schedule_padding+0xf7)[0x55a1ec82b6e7]
/usr/bin/tor(circpad_machine_spec_transition+0xe5)[0x55a1ec82afd5]
/usr/bin/tor(circpad_cell_event_nonpadding_received+0x7a)[0x55a1ec82b2da]
/usr/bin/tor(circpad_handle_padding_negotiate+0x160)[0x55a1ec82ce10]
/usr/bin/tor(+0xbb496)[0x55a1ec853496]
/usr/bin/tor(circuit_receive_relay_cell+0x490)[0x55a1ec855b80]
/usr/bin/tor(command_process_cell+0x3f8)[0x55a1ec837958]
/usr/bin/tor(channel_tls_handle_cell+0x39b)[0x55a1ec816c8b]
/usr/bin/tor(+0xa84dc)[0x55a1ec8404dc]
/usr/bin/tor(connection_handle_read+0xa0d)[0x55a1ec8040fd]
/usr/bin/tor(+0x7134e)[0x55a1ec80934e]
/usr/lib/x86_64-linux-gnu/libevent-2.1.so.6(+0x1e8f8)[0x7fb52a4ad8f8]
/usr/lib/x86_64-linux-gnu/libevent-2.1.so.6(event_base_loop+0x53f)[0x7fb52a4ae33f]
/usr/bin/tor(do_main_loop+0xfd)[0x55a1ec80a65d]
/usr/bin/tor(tor_run_main+0x126d)[0x55a1ec7f815d]
/usr/bin/tor(tor_main+0x3a)[0x55a1ec7f54da]
/usr/bin/tor(main+0x19)[0x55a1ec7f5069]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7)[0x7fb528ddcb97]
/usr/bin/tor(_start+0x2a)[0x55a1ec7f50ba]
</pre>

## Further work

The padding likelyhoods chosen for the RBP machines were inspired by our results from the original paper, but the values for this implementation have not been rigorously tested. Further work should be done to identify ideal configurations for this machine.
