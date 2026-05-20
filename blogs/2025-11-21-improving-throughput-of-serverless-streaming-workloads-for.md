---
title: "Improving throughput of serverless streaming workloads for Kafka"
url: "https://aws.amazon.com/blogs/compute/improving-throughput-of-serverless-streaming-workloads-for-kafka/"
date: "Fri, 21 Nov 2025 20:02:57 +0000"
author: "Anton Aleksandrov"
feed_url: "https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/feed/"
---
Event-driven applications often need to process data in real-time. When you use AWS Lambda to process records from Apache Kafka topics, you frequently encounter two typical requirements: you need to process very high volumes of records in close to real-time, and you want your consumers to have the ability to scale rapidly to handle traffic spikes. Achieving both necessitates understanding how Lambda consumes Kafka streams, where the potential bottlenecks are, and how to optimize configurations for high throughput and best performance.
