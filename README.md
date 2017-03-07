#Stateless REST Services and Real-Time Concurrent User Metrics
Sky Schulz, March 6 2017

In the remaining weeks leading up to launch, the product owner asked the team when the concurrent user metrics dashboard would be ready, to which we responded "uh, we'll get right on that". This was a surprise request that had not been previously reviewed or scoped. I volunteered to sort the details, while the rest of the team worked on fine tuning and last minute bug fixes of existing services.

Luckily, there was already some tooling in place that could be leveraged with, hopefully, not too much effort: authentication tokens. A unique identifier present in virtually every service request.

The service layer is a stateless, JSON over HTTP, remote procedure call, multi-tenant, shared service architecture, with a proprietary sub-protocol. Written in PHP and running on a fleet of Linux virtual machines, with nginx providing HTTP termination and FastCGI routing, the service layer is backed by a cache cluster in Redis, and persistence cluster in MongoDB.

Every service request, aside from authentication requests, must include a valid authentication token. The token itself being a key for the authentication object, produced during authentication, and stored in Redis, with a two hour expiration. A token is consider expired if it cannot be retrieved from Redis, and any request with an expired token is ignored.

In addition, the client is configured to send keep alive requests every 60 seconds. If keep alive requests fail too many times, the client attempts to authenticate again. This behavior would later contribute to catastrophic DDoS events. But that's a story for another time. For now, it will suffice to know that all active clients will have had communication with the service layer, whether a specific service request or a keep alive request, in any given minute.

Since the authentication token is present in every request, I suggested we count every observed authentication token within a sample time window, and publish the sample size to Elasticsearch (the time-series database storing our metrics). This strategy was agreed to by the team, and I set to work.

I added a new class, Capacity, for mediating the token sampling and metrics publishing. The class was responsible for storing all observed token keys in a Redis set. One set per sample window (Observed Sample Set). The sample window being of fixed size (e.g. 120 seconds), each complete sample interval would, in theory, contain 100% of the active authentication tokens. Every 120 seconds, a new Observed Sample Set would be created in Redis, with a one hour expiration. The sets would automatically expire out of Redis, requiring no additional maintenance.

The Capacity class also provided a simple API to publish the metrics of the current, and preceding Observed Sample Sets. Given a 0-based index, n, it would read the size of the nth (0 == current set, 1 == previous set, ...) Observed Sample Set from Redis, and publish it to Elasticsearch.

I then modified the common authentication validation path, integrating the sampling function of the Capacity class at both authentication and authorization phases. Capturing tokens at initial creation and  helped ensure the broadest coverage in the sample set.

A task was configured to run every 15 seconds that would call the publish operation of the Capacity class for the current (in-progress) and previous (complete) Observed Sample Sets. 

The respective counts could then be projected to dashboard-style graphs using Kibana, providing the desired near real-time concurrent user metrics.

---
Copyright © 2017 Sky Schulz. All rights reserved.