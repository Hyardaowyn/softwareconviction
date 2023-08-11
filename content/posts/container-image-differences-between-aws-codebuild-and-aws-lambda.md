---
title: "Things you need to know before deploying a python software package that was built on AWS Codebuild and deployed on AWS Lambda"
date: 2023-08-14T14:30:22+02:00
draft: true
---

# Context
pip is the official python package manager.
poetry and pipenv, are some other well known package managers
But if you work with PyPI you will use a package manager that uses pip under the hood.
## Domain Logic with Environment Dependencies

Domain logic should ideally remain devoid of environment variables or any other environmental or infrastructural concerns.
When testing the domain, infrastructural or initialization matters are generally disregarded.
This can lead to their omission in unit tests, particularly when default environment variables are in play.
In practice, multiple code branches are often created, yet they may not all be adequately tested.
This becomes problematic when it involves critical functionality, as the exhibited behavior on production might differ from that in testing, potentially leading to severe consequences.
It's disconcerting if the behavior in production significantly varies from that in non-production/test environments.
This divergence indicates a likely misalignment in modeling domain concerns.

# Real-life Context

In our use case/application service, we managed collections of files. To prevent an indefinite growth of files, a housekeeping process was established. This process involves selecting collections for deletion based on specific criteria, including:
- Collection creation date
- Collection name beginning with a special character
- Configured end-of-use date for collections
- Collection-wide parameter defining post-end-of-use retention duration

Furthermore, some collections must never be deleted due to their usage in testing. This extra requirement is relevant only in a development/test context.

## What Went Wrong

No tests were conducted with environment variables configured to mimic the production environment. This oversight in the test suite resulted in incorrect behavior in the production environment.

## Initial Solution

Environmental-specific unit tests were added to the housekeeping use case.
However, this solution necessitates awareness of the environment variable dependency, which can be forgotten in the future or overlooked during development.
This environmental dependency should raise concerns, indicating something might be amiss or missing.

## A Better Solution

Typically, a model should remain conceptually unaffected by test concerns, since it is driven by the domain experts.
Nevertheless, the concept of preventing the deletion of certain collections aligns with the mental model.
Including this parameter as part of the domain concept—i.e., the collection—can be seen as potentially useful in both production and testing scenarios.
Although, admittedly, it wasn't initially a part of the ubiquitous language.
We chose this approach for its numerous advantages.
Now, whether an issue can be deleted or not is encapsulated within the issue itself.
It has become a domain concern devoid of environmental configuration.
Had we placed the logic concerning the deletion of specific issues in a secondary adapter, changes to the secondary adapter might have inadvertently impacted this logic, resulting in the erroneous removal of collections that should have been retained.
This risk is further magnified in the future when a new team might not grasp the intricacies of the "business logic in the secondary adapter."

# Conclusion

Do not put environmental concerns in the domain/core. Try to identify an omission in the conceptual model and try to further enhance it.
Doing this will result in better and more readable tests.
A better grasp of the used model, also leads to less future mistakes.