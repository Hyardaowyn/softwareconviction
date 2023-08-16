---
title: "The pitfall of business logic in secondary adapters"
date: 2022-04-07T18:28:22+02:00
draft: true
---

# Context
hexagonal architecture strives to design your application in such a way that you core domain is isolated from infrastructure concerns.
Things like event/message structure
# Business logic in a secondary adapter

# secondary adapter is an adapter for a detail, it should be dumb 
A secondary adapter is an adapter for an infrastructure detail. Ideally it should be as dumb as possible.

# what if you switch the secondary adapter, it's only a secondary adapter right?
# all unit tests pass
# alternatives that i do not prefer
## e2e all the thing
e2e test everything in the pipeline (yes it covers your bases, but impacts RTO and development a bit too much)
## integration test all the things

# corollary
Do not use stored procedures (with business logic) in combination with hexagonal architecture.

# conclusion 
put all filter/ selection logic in use case