---
date: 2022-10-21
image: 'post/2022/Mongodb.png'
title: 'State machine using MongoDB changestreams'
slug: state-machine-using-mongodb-changestreams
toc: false
tags:
  - Cloud
  - MongoDB
---

## Introduction

I worked on a project where we needed to implement a process that spans multiple days. The process is dependent on the internet connectivity of our customers IoT device.

Usually I would go for something like [temperol.io](https://temporal.io/), Azure Durable Entites or AWS Step Functions. But incorperating new infrastructure is always a challenge and the project wanted to keep the infrastructure as simple as possible. Meaning re-using existing infrastructure.

That narrowed the options down to kubernetes and MongoDB =)

## Bulding a state machine with MongoDB and kubernetes

In this post I will show how to build a state machine using [MongoDB changestreams](https://docs.mongodb.com/manual/changeStreams/) and kubernetes. The state machine will be used to track the progress of a process that spans multiple days. 

**Each state** is represented as a column in a single document. The document is updated by the state machine when the process moves to the next state.

**Each state** in the state machine is implemented as a kubernetes **statefulset**. Why statefulset? Because we need to store a [resume token](https://www.mongodb.com/docs/manual/changeStreams/#std-label-change-stream-resume-token) in a durable place. The resume token is used to resume the changestream when the statefulset is restarted (update or crash).

## Process manager

I need a process manager that takes in my state business logic and executes it. The process manager needs to keep track of the resume token and make sure its persisted in case of a crash or update. This is what I came up with:


```ts
import fs from 'fs';
import { ChangeStreamDocument, Document, MongoDatabase } from '@lib/mongo';
import { config } from '@lib/config';
import log from '@lib/logger';

interface workInterface {
  (item: ChangeStreamDocument<Document>): Promise<void>;
}

export class Process {
  private filter: any[];
  private mongo: MongoDatabase;
  private isShutdown = false;
  private lastEvent: { init: boolean; token: unknown };
  private cursorLocation: string;
  private work: workInterface;

  constructor(filter: any[], cursorFileName: string, work: workInterface) {
    this.filter = filter;
    this.work = work;
    this.mongo = new MongoDatabase(config());
    this.cursorLocation = '/data/' + cursorFileName;
    try {
      this.lastEvent = JSON.parse(fs.readFileSync(this.cursorLocation).toString());
    } catch {
      this.lastEvent = { init: true, token: {} };
    }
  }

  private async start() {
    await this.mongo.initDatabase();
    const cursor = this.mongo
      .getDb()
      .collection(config().mongodb.downgradeCollection)
      .watch(this.filter, this.lastEvent.init ? {} : { resumeAfter: this.lastEvent.token });

    try {
      while (!this.isShutdown) {
        const data = await cursor.next();
        this.lastEvent.token = data._id;
        await this.work(data);
      }
    } catch (err) {
      log.error(err);
      fs.writeFileSync(this.cursorLocation, JSON.stringify(this.lastEvent));
      process.exit(1);
    }
  }

  public async run() {
    const signals = ['SIGTERM', 'SIGINT', 'SIGQUIT'] as const;
    signals.forEach((signal) => {
      process.once(signal, () => {
        this.isShutdown = true;
        if (Object.keys(this.lastEvent.token).length > 0) {
          this.lastEvent.init = false;
        }
        fs.writeFileSync(this.cursorLocation, JSON.stringify(this.lastEvent));
        log.info(`Received ${signal}, will resume after ${JSON.stringify(this.lastEvent)}`);
        process.exit(0);
      });
    });
    await this.start();
  }
}
```

The process manager takes in a filter, a cursor file name and a work function. The work function is the business logic that is executed when a document is updated. 
The process manager listens to termination signals and saves the resume token to a file. The resume token is used to resume the changestream when the process is restarted.

In case of an error in the work function the process manager will save the resume token and exit. This will cause the statefulset to restart and resume the changestream. It is important that you implement your business logic in a way that it can be restarted. In other words, the business logic should be idempotent. 


Lets look at an example of how to use the process manager:

```ts
import { ChangeStreamDocument, Document, MongoDatabase } from '@lib/mongo';
import { config } from '@lib/config';
import log from '@lib/logger';
import { Process, states } from './process';

const filter = [
  {
    $match: {
      operationType: 'insert',
    },
  },
];

const work = async (data: ChangeStreamDocument<Document>) => {
  const now = new Date().toISOString();
  const { operationType } = data;
  if (operationType !== 'insert') {
    log.warn(`Unexpected operation type: ${operationType}`);
    return;
  }

  const collection = MongoDatabase.getDb().collection(config().mongodb.downgradeCollection);

  const { fullDocument: item } = data;
  if (item.state !== states.started) {
    log.warn(`State is not ${states.started} but ${item.state}`);
    return;
  }

  const { _id, state, user } = item;
  log.info(`Processing ${_id} with state ${state} for user ${user}`);
  
  /. . . business logic . . ./


 // Move to next state
  const nextState = states.downloading;

  await collection.updateOne({ _id }, { $set: { state: nextState, updatedAt: now } });
};

const started = new Process(filter, `started.json`, work);
started.run();
```

I call this the "started" state and will be deployed as it own statefulset in kubernetes. The statefulset will be configured to restart on failure.


With this setup I can add as many states as I want. Each state will be deployed as a statefulset in kubernetes.
For my use case I needed 5 states. Each state did one thing and then moved process to the next state.


## Conclusion

I hope this post was useful. If you have any questions or comments you can reach me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev).