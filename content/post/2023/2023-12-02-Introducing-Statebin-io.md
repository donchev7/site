---
date: 2023-12-02
image: 'post/2023/statebin2.png'
title: 'Introducing Statebin.io'
slug: 'statebin-io'
toc: true
tags:
  - DevOps
  - Serverless
  - SaaS
---


I have been working on a new project called [Statebin.io](https://statebin.io). It is a SaaS tool (free) that provides atomic counters, persistent storage, and locking mechanisms. It is a versatile tool designed to simplify and streamline various data management tasks. It offers a range of functionalities that cater to the needs of developers, system administrators, and tech enthusiasts alike. Think of it as httpbin.org but for state management :)


## Features

Currently, Statebin.io offers the following features. More features are planned for the future. If you have any suggestions, please let me know!

### Atomic Counters

Atomic counters are used to track the number of times an event has occurred. For example, you can use them to track the number of times a user has logged in to your website or the number of times a particular API has been called. Statebin.io provides a simple API that allows you to create and increment atomic counters. You can also use the API to retrieve the current value of a counter.

Use cases I use it for:

- CI Pipelines: Keeping a precise track of the number of builds, ensuring accurate build sequencing and management. I know Azure DevOps provides BuildIDs but its a global counter. I needed a counter that is unique to each pipeline / branch.

- GitHub Profile View Tracking: By leveraging the counter functionality, I can easily track the number of views on my GitHub profiles and repositories. I'll be writing a blog post on how I did this soon.
 
- Link Click Tracking: Haven't implemented this yet but I can see how this can be easily implemented using the counter functionality.

### Persistent Storage

Statebin.io provides a simple API that allows you to store and retrieve data. You can use this API to store and retrieve any type of data, including JSON, XML and binary data. The API is simple with GET, DELETE, and PUT requests.

Use Cases:

- Settings files / Config files
- Coverage SVG Files
- CI Pipeline Files


### Locking Mechanisms 

Locks have a similar API to Atomic Counters. You can acquire and release locks. Locks by default have a TTL (Time To Live) of 60 minutes.

Use Cases:

- Resource Locking in CI Pipelines
- Exclusive Access Management


## The Architecture

I created statebin.io using a serverless architecture. I used cloudflare workers that run on top of the cloudflare edge network. The workers are written in TypeScript and are deployed using the wrangler CLI tool. The workers are backed by more cloudflare, namely durable objects. Durable objects are a new serverless primitive that allows you to store stateful, long-lived objects in the cloudflare edge network.

For persistent storage, I use R2 with custom metadata detection. Currently the storage is limited to 100MB per object.

The combination of these technologies ensures that Statebin.io is not only globally available but also maintains high reliability. Data integrity and uptime are paramount, making it a dependable choice for your data management needs.

## Conclusion

I hope you find Statebin.io useful. I have been using it for a while now and it has been working great. I have a few more features planned for the future.

Go check it out at [Statebin.io](https://statebin.io) and let me know what you think!