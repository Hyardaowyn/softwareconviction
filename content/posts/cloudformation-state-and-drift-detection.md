---
title: "Cloudformation drift detection"
date: 2022-05-18T17:10:22+02:00
authors: ["Geert Van Wauwe"]
draft: false
---

CloudFormation, undoubtedly, has proven to be an effective tool for orchestrating and managing infrastructure as code (IAC). However, one of the challenges it faces is the assurance of maintaining the accurate synchronization between the defined IAC state and the actual deployed state. This is where Terraform stands out, leveraging its capability to interact directly with APIs, enabling a more precise alignment between the intended state and the live resources.

Terraform's advantage in making API calls offers a tangible benefit in scenarios where granular control over resources is essential. This approach enables quicker, more targeted updates and helps avoid any potential discrepancies that might arise from the intermediary processes in CloudFormation.

Yet, it's important to note that CloudFormation's value isn't diminished, especially when the assessment and resolution of drift are straightforward endeavors. If identifying disparities between the desired and existing states is simple and rectifying them doesn't pose a significant challenge, CloudFormation remains a viable choice.

However, herein lies the crux of the matter. The complexities emerge when addressing specific nuances in drift detection and managing changes. For instance, consider the challenge of bugs surfacing during drift detection, such as those related to the BucketKeyEnabled property within the configuration of an S3 bucket's Server-Side Encryption (SSE). Such intricacies can potentially introduce unexpected behavior and demand meticulous attention during the drift assessment phase.

Furthermore, the application of tags to older log groups might not unfold as expected, adding another layer of complexity. Similarly, the requirement for the visibility timeout to exceed the lambda timeout, despite this not being enforced three years ago, necessitates a careful approach, potentially involving a rolling update to ensure compliance.

In spite of these problems, CloudFormation maintains its status as a reliable go-to option for orchestrating infrastructure. Its comprehensive approach to managing resources as code has been embraced by countless users. However, it's crucial to acknowledge that, like any tool, CloudFormation is not immune to imperfections and challenges. As your preferred choice, it continues to offer immense value, but it's important to approach its utilization with an understanding of its limitations.

It really depends on the development effort that Amazon is willing to invest in improving the tool.
Though it must be said that the provided improvements have been insufficient.
Therefore I tend to prefer terraform for setting up new projects nowadays.
No service will be delivered without a CLI API, so new services can be onboarded in Terraform quite quickly.
It also gives you more control than cloudformation currently provides, though the mindset is a bit different regarding roll back and roll forward strategy.


In conclusion, the landscape of infrastructure as code is marked by the strengths and limitations of various tools.
While Terraform excels in precision and resource control through direct API interaction, CloudFormation's value persists in its comprehensive approach.
The intricate challenges of drift detection, resource changes, and historical inconsistencies remind us that while these tools are powerful allies, thoughtful and adaptable strategies are key to effective infrastructure management.
