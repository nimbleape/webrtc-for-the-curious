---
title: Trickle ICE
type: docs
weight: 99
---

# (TODO) Intro to Trickle ICE
...

# Trickle ICE and VOIP servers
VOIP servers, unlike normal WebRTC ones, typically can’t do trickle ICE. As a consequence, during the signalling phase (CHECK), the client can’t start sending local candidates to the other side as soon as it gathers them. Instead, it needs to collect them all and add them to the SDP. When it's finished gathering, the client sends the SDP containing the list of all possible candidates to the other peer. Browsers have a 40-second timeout for this process, at the end of which gathering is stopped and the SDP is sent with the candidates collected until that point.

One of the possible reasons for hitting this timeout could be that one of the routes on the machine’s connection setup can’t reach the internet, for example. In this case, the user would be left waiting for 40 seconds before connecting to the VOIP call: hardly an ideal user experience!

Therefore, when dealing with VOIP, it is highly recommended to put some logic in place that will stop the ICE candidate gathering after a good number has been collected, and send the SDP. What constitutes a "good" number of candidates depends on the application and should be tailored on each case. As a bare minimum, one `srflx` candidate should make the connection possible in the most common cases.

How do you stop gathering candidates? That depends on the client VOIP library in use. In the case of JSSIP, for example, you would call `ready()` on the `icecandidate` event:

```js
rtcSession.on('icecandidate', (evt) => {
  // Determine if we want to stop gathering here
  // ...
  evt.ready();
});
```

Evidently, there are compromises to be made: When sending the SDP with only one candidate, we're essentially putting all our eggs in one basket. On the other hand, the more the candidates, the longer the wait for servers (especially TURN servers) to reply with candidates.

## Example
In general, it's safer to collect both `srflx` and `relay` candidates to be relatively sure the connection can be established even with strict NAT setups. Since TURN servers are generally slower to come back with candidates, this approach does mean we’re waiting longer than just for `srflx`. The increase wait will probably be in the range of 300-400 ms per `relay` candidate (or per TURN service? or total?? CHECK). Whether the wait is worth it ultimately depends on the application.

### The problem
What happens, if one of the STUN or TURN services is unavailable? The client will keep waiting for the missing candidate type until the browser steps in and stops ICE gathering after 40 seconds. In practice, the user probably won't wait that long and will try re-initiating the call. If the service is still unavailable, however, they would still need to wait the 40 seconds to be able to connect. Again, we're providing a bad user experience.

We could be tempted, then, to put the collection of candidate types in an `OR` condition. That is, cut the ICE candidates gathering after we have at least one `srflx` or one `relay` candidate. This should cover most network scenarios and, if one service is unavailable, the client won't be stuck waiting. In practice, STUN servers are almost always quicker to return a candidate than TURN servers, therefore our condition will almost always be satisfied by a `srflx` candidate. This type of candidate will still work on simple NATS but, if a user is behind a strict NAT and is trying to connect with only `srflx` candidates, the call will never connect. Chances are, because STUN servers generally reply with a candidate faster than TURN servers, that this user will *never* be able to make any calls!

What about defaulting to only `relay` candidates then? This solution, however possible, comes at the cost of usage of TURN networks. (LINK TO CHAPTER) Because TURN servers actually transfer the data throughout the call, TURN server providers will charge for the cost of the ingress and/or egress bytes. Since only a small percentage of overall connections actually do require a TURN network, this solution would be overkill in most scenarios. There are, however, cases where this approach would make sense, for example when handling connections from a corporate network with very strict firewall settings.

### The solution
As a general rule, when working with VOIP servers, a good starting point is to collect (at least) one `srflx` AND one `relay` candidate before stopping the gathering phase. This means the wait to connect is a bit longer than just for a `srflx` candidate, due to the slower response time of typical TURN servers. The increase wait will probably be in the range of 300-400 ms (but the overall wait is the same as when only accepting a `relay` candidate, as in the last example above). In most cases, the extra wait is worth it, if it means the user will definitely be able to make a call (and it’s virtually unnoticeable).

As a safeguard, we should also add a timeout, for example of 500-1000 ms, after which the OR condition is applied. This short-circuits the event in which one of the services is unavailable and still allows the majority of users to connect.

The following is an example using JSSIP:

```js
let candidateTypes = {
  srflx: 0,
  relay: 0,
};
let fallbackTimer = null;

rtcSession.on('icecandidate', (evt) => {
  // Extract the candidate type from the candidate string
  let type = evt.candidate.candidate.split(' ');
  candidateTypes[type[7]]++;
  if (!fallbackTimer) {
    // Set the short-circuit fallback
    fallbackTimer = setTimeout(() => {
      if (candidateTypes['srflx'] >= 1 || candidateTypes['relay'] >= 1) {
        fallbackTimer = null;
        evt.ready();
      }
    }, 500);
  }

  // Accept at least one relay and one srflx candidate
  if (candidateTypes['srflx'] >= 1 && candidateTypes['relay'] >= 1) {
    clearTimeout(fallbackTimer);
    fallbackTimer = null;
    evt.ready();
  }
});
```

There are obviously many solutions to this problem, so each application will benefit from a bespoke strategy. What’s important is to make sure there is a strategy in place, however simple, when working with VOIP servers.
