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
                                                      PAD SND  <-----^
                                                               |     |
+-------------+            +-------------+         +-------------+   |
|             | NONPAD RCV |             |         |             |--->
|             |    or      |             | PAD SND |             |
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

For full details, see the code and docs in the source files.

## Issues

Since this padding machine heavily relies on 0usec delays to immediantly trigger burst padding, this machine is affected by [issue #31653](https://trac.torproject.org/projects/tor/ticket/31653). The bandaid solution suggested by pulls is used so that the REB machine can function for the time being. 

Strangely, the RBB machine continues to fail despite this solution. The circuit on the relay-side continues to crash with the following error:

<pre>

</pre>

## Further work

The padding likelyhoods chosen for the RBP machines were inspired by our results from the original paper, but the values for this implementation have not been rigorously tested. Further work should be done to identify ideal configurations for this machine.
