#Stateless REST Services and Real-Time Concurrent User Metrics
Sky Schulz, March 6 2017

After spending several months preparing the service layer for launch, deploying load generators to 500 AWS instances, optimizing common code paths, and introducing judicious caching, it was determined that real-time concurrent user metrics would also be required.

The service layer is a stateless, JSON over HTTP, remote procedure call, multi-tenant, shared service architecture, with a proprietary sub-protocol. Written in PHP and running on a fleet of Linux virtual machines, with nginx providing HTTP termination and FastCGI routing, the service layer is backed by a cache cluster in Redis, and persistence cluster in MongoDB.

Every service request, aside from authentication requests, must include a valid authentication token. The token itself being a key for the authentication object, produced during authentication, and stored in Redis, with a two hour expiration. A token is consider expired if it cannot be retrieved from Redis.

In addition, the client is configured to send keep alive requests every 60 seconds. If keep alive requests fail too many times, the client attempts to authenticate again. This behavior would later contribute to catastrophic DDoS events. But that's a story for another time. For now, it will suffice to know that all active clients will have had communication with the service layer, whether a specific service request or a keep alive request, in any given minute.

After a bit of brainstorming, it was determined that the authentication token sent in every request could be leveraged in calculating the desired (near) real-time concurrent user metrics.

If the only keys present in Redis were access tokens; and, if the expiration of those keys were small enough, a count of active keys in Redis could provide a close enough approximation to call it good. Unfortunately, the same Redis cluster was used to store many different kinds of objects. And the expiration of the authentication tokens was at least 2 hours: a user could have signed in and been active for just 10 minutes and never return, but the access token will persist in Redis for another 110 minutes. Just counting keys was out.

It was decided to count observed authentication tokens within a “sliding window”, and project those counts to the Elasticsearch metrics store. This was achieved by modifying the common authentication validation path and storing all observed token keys in a Redis set. One set per sample window (Observed Sample Set). The sample window being of fixed size (e.g. 120 seconds), each complete sample interval would, in theory, contain 100% of the active authentication tokens. Every 120 seconds, a new Observed Sample Set would be created in Redis, with a one hour expiration. The sets would automatically expire out of Redis, requiring no additional maintenance.

A task was configured to run every 15 seconds that would read the count of the current (in-progress) and previous (complete) Observed Sample Sets from Redis and send those metrics to Elasticsearch. The respective counts could then be projected to dashboard-style graphs using Kibana, providing the desired near real-time concurrent user metrics.

In retrospect, it may have been better to use a single sorted set in Redis, with the sort value being time and the set key being the authentication token. It would then be possible to query the sorted set for the count of any given window period (e.g. 1 minute, 5 minute, 15 minute, etc.). However, additional maintenance would be required to purge older keys periodically, otherwise the set would grow unbounded.

———
Copyright © 2017 Sky Schulz. All rights reserved.