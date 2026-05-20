---
title: "Building responsive APIs with Amazon API Gateway response streaming"
url: "https://aws.amazon.com/blogs/compute/building-responsive-apis-with-amazon-api-gateway-response-streaming/"
date: "Wed, 19 Nov 2025 23:10:51 +0000"
author: "Anton Aleksandrov"
feed_url: "https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/feed/"
---
Today, AWS announced support for response streaming in Amazon API Gateway to significantly improve the responsiveness of your REST APIs by progressively streaming response payloads back to the client. With this new capability, you can use streamed responses to enhance user experience when building LLM-driven applications (such as AI agents and chatbots), improve time-to-first-byte (TTFB) performance for web and mobile applications, stream large files, and perform long-running operations while reporting incremental progress using protocols such as server-sent events (SSE).
