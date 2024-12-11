---
title: "Survey: Container Registries – 1. Edition"
draft: true
date: 2024-12-11T18:20:35+02:00
tags:
- docker
- registry
- secrets
---

I got the chance to get access to the scan results from [Chrisopher Dreher a.k.a Schniggie](https://twitter.com/schniggie) and his research on container registries. I took his results as an input for my research, downloaded all images, extracted and verified secrets and performed some statistics on the data.  

<!--more--> 

## Introduction

Have you ever wondered what people upload to self-hosted Docker registries? If so, you're not alone – I’ve often been curious about this myself! Fortunately, [Schniggie conducted a scan of public IP ranges from cloud providers some time ago and uncovered a number of open Docker registries](https://dreher.in/blog/unprotected-container-registries). With access to Schniggie's findings, I had the opportunity to perform my own analysis.

Using the provided list, I checked which of the identified hosts were still accessible without requiring credentials. To my surprise, 4,090 servers were still open, allowing me to list the images stored in their registries. Rather than adding new images, I chose to download the existing image layers and scan them for valid secrets.

Secrets, such as API keys and passwords, are fundamental to modern systems, enabling secure access to infrastructure, applications, and sensitive data. When exposed, these secrets can pose significant risks, allowing malicious actors to cause harm to individuals, organizations, and systems.

In this article, I’ll walk you through how I scanned these registries, analyzed the dumped layers, and identified secrets within them. Finally, I’ll share the key findings from my analysis.

I evaluated my results a few months after I dumped the registries and scanned for secrets. Sadly, I realized that I missed implementing the logic around Docker Manifest Schema Version 2. So the analysis results are only for images that are published with Docker Manifest Schema Version 1. I will follow-up with a 2. Edition post as soon as I have fresh scan results from Schniggie and implemented support for Docker Manifest Schema Version 2.

## Approach & Methodology

The input data from Schniggi included masscan results, containing IP addresses, ports, and server response headers. To simplify the process, I filtered the data by identifying response headers with `Docker-Distribution-Api-Version: registry/2.0`, which is the default header returned by the Docker [registry](https://hub.docker.com/_/registry) image. Using this filtered list of target servers, I followed a systematic approach for each server:

1. **Verify Accessibility:** Check if the registry server is still online and responds with the appropriate registry header.
2. **List Available Images:** Retrieve a list of all available images in the registry.
3. **Retrieve Tags:** Fetch the published tags for each image.
4. **Fetch Manifests:** Obtain the manifest associated with each tag.
5. **Download Layers:** For each layer referenced in the manifest:
    - If the layer does not already exist locally, download and store it in a central storage location.
    - Link the layer to its corresponding server, image, and tag.

After all layers were downloaded, I analyzed them using Trufflehog to extract and validate secrets.

**Efficient Layer Management**

A key advantage of Docker is its use of unique layer hashes, ensuring that identical layers across different images do not need to be downloaded or analyzed multiple times. For example, if an image references `ubuntu:20.04`, its layers are identical across registries. Once analyzed, these layers do not require reprocessing, saving both time and storage.

**Post-Processing**

Once the analysis of a registry was complete:

**Results were stored securely**

Local copies of the analyzed layers were deleted to free up space.
This iterative process was repeated for each registry, leveraging the deduplication of layers to streamline the workflow.

**Manifest Analysis**

An additional step involved analyzing manifest.json files, as secrets can also be inadvertently stored in these files. Using Trufflehog, I scanned these manifests for any valid secrets after dumping all registries.

## Tooling
Interacting with untrusted Docker registries via the local Docker daemon can be cumbersome and potentially risky. To address this, I developed an API client called **reggidump**, which simplifies the process by directly interacting with Docker registries and downloading image layers to the local filesystem.

**reggidump** is open-source and available on GitHub under [NodyHub/reggidump](https://github.com/NodyHub/reggidump). The repository also includes shell scripts for post-processing, located in the scripts directory. These scripts streamline tasks such as layer analysis and result organization, further enhancing the efficiency of the workflow.

Sadly, I started my data analysis a bit late and realized that I missed half of the data set, bc/ the implementation of manifest.json version was error-prone. So, potentially, there were way more secrets available, but that's something for a re-run after Schniggie has scanned again the public IP ranges.

## Results

After completing the analysis, the following results were obtained:

### Key Metrics

- **Dumped Registries:** 4.088
- **Images:** 189.230
- **Dumped manifest.json v1:** 81.644
- **Analyzed Layers:**  265.608
- **Valid Secrets Identified:** 896
- **Secrets Found in Manifests:** 12
- **Detectors Used:** 76

896 secrets were identified across 4,090 registries, meaning that approximately 1 in 5 registries contained sensitive information, which is a lot! The high number of layers (265.608) underscores the extensive use of Docker for containerized applications. 

### Averages and Probabilities

- **Average Layer Size:** 1.21 MB
- **Maximum Layer Size:** 548 MB
- **Minimal Layer Size:** 4.0 byte
- **Average Layers Per Image:** 19,87
- **Probability of a Secret in a Registry:** 908/4.088 =  0.2221
- **Probability of a Secret in an Image:** 908/81.644 = 0.0111
- **Probability of a Secret in a Layer:** 908/265.608 = 0.0034
- **Probability of a Secret in a Manifest:** 12/81.644 = 0.0001

### Top Insights

- Top 5 Image Names: 
   1. `autoquote-app`: 1752
   2. `library/alpine`: 1291
   3. `activities-app`: 1291
   4. `com.elbi.backend`: 778
   5. `live`: 743
- Top 5 Tag Names: 
   1. `latest`: 9929
   2. `1.0.0`: 498
   3. `1.0`: 494
   4. `dev`: 385
   5. `v1`: 289
- Top 5 Architectures: 
   1. `amd64`: 79958
   2. `arm64`: 1451
   3. `arm`: 176
   4. `386`: 30
   5. `s390x`: 24
- Top 10 Secret detectors:
   1. GCP: 149
   2. AWS: 111
   3. TelegramBotToken: 61
   4. OpenAI: 55
   5. PrivateKey: 54
   6. MongoDB: 48
   7. Alibaba: 41
   8. SendGrid: 30
   9. SlackWebhook: 28
   10. Github: 25

### Visualizations

Visualization is not my passion and so my capabilities to plot huge graphs. If someone is interested in plotting the image layer network, feel free to [download the dot](/posts/2024-12-11_graph.dot.gz) file and play around with it 🙂

## Next Steps for the Security Community

The security community must prioritize improving the Docker ecosystem to ensure container registries are **secure by default**. This involves mandatory **authentication mechanisms** for access, reducing the risk of unauthorized usage and exposure.

Another critical area for improvement is the **management of secrets**. While the **12-factor app methodology** provides a solid foundation, its principles must be more widely adopted. Developers and users need accessible resources, practical training, and continuous education to embed these practices into their workflows effectively.

## Ethical Challenges and Collaboration

Through this analysis, I’ve uncovered numerous exposed secrets, which raises an important ethical question: How can we responsibly notify the owners while protecting their confidentiality? This is a delicate task that requires careful consideration of ethical, legal, and practical factors.

If you have suggestions or experience in handling situations like this, I would greatly appreciate your collaboration. Please feel free to reach out at **jan@nody.cc**. Together, we can work toward building a safer, more secure container ecosystem.

## Ethical Remarks

Throughout this project, I have adhered to responsible disclosure practices by refraining from publicly sharing specific vulnerabilities or sensitive information. My focus has been on highlighting broader security risks and the potential impact of misconfigured container registries.

To protect the privacy of individuals and organizations, all data has been anonymized, and no personal or identifiable information has been disclosed. The primary aim of this work is to raise awareness about the security challenges in open container registries and to promote best practices for securing container environments.

I am committed to collaborating with the community to address these vulnerabilities and improve the security of container ecosystems. It is also important to stay mindful of local laws and regulations regarding data privacy and security. While public registries are technically accessible to anyone, obtaining explicit consent from registry owners is an ethical best practice, particularly if detailed findings are to be shared.

Finally, I urge others to handle these findings responsibly. Avoid sensationalizing risks or exaggerating potential impacts, and consider the broader implications for the affected individuals and organizations.
