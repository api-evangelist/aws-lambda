---
title: "Serverless strategies for streaming LLM responses"
url: "https://aws.amazon.com/blogs/compute/serverless-strategies-for-streaming-llm-responses/"
date: "Fri, 21 Nov 2025 03:42:56 +0000"
author: "KyungYong Shim"
feed_url: "https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/feed/"
---
Modern generative AI applications often need to stream large language model (LLM) outputs to users in real-time. Instead of waiting for a complete response, streaming delivers partial results as they become available, which significantly improves the user experience for chat interfaces and long-running AI tasks. This post compares three serverless approaches to handle Amazon Bedrock LLM streaming on Amazon Web Services (AWS), which helps you choose the best fit for your application.
