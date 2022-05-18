---
title: "Slashing AWS RDS costs for a database instance in an enterprise context"
authors: ["Geert Van Wauwe"]
draft: true
---
cloudformation works great, yet it is hard to see whether the IAC state is the current state.
This is where terraform takes the advantage with api calls.
however If it is easy to check and easy to fix drift, then cloudformation is fine
here lies the rub. 
bugs in driftdetection and change
e.g.BucketKeyEnabled property of S3 bucket SSE.
tags not applies to older loggroups.
visibility timeout must be greater than lambda timeout, yet this was not enforced 3 years ago, please do rolling update.
Cloudformation is still my goto but it's not perfect.