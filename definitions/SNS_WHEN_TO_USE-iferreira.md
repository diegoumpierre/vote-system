# AWS SNS and when to use

## Overview
Amazon Simple Notification service or SNS, is a message service for applications, can be managed to notify pusblishers (apps) and subscrivbers (apps).

![alt text](./images/sns-arch2.webp)



## When to use
1. Fan-Out Pattern
Fanout Pattern refers to a system design approach for efficiently distributing data to multiple consumers in a scalable manner.

![alt text](./images/fanout.png)

2. Real time notification
  * Alert Systems
  * Notification, (email, sms, push notification)
  * Webhooks to external software

3. Event Broadcaster
A single event must be delivered to different places

4. Micro Service decoupler
When services needs to react to events even don't knwoing each other

5. Mult-Region / Mult-AWS-Accounts
As sub-title says, propagate events to different regions or AWS accounts

## WHen NOT to use SNS
* Sequential proccess
* Only one or few consumers
* Complex retry order (for it, sns + sqs is a must on)
* Big messages, SNS can deliver max 256kb

## SNS in Deep


### Core components

- Publisher
- Topic
- Subscriber

#### Pusblisher

```jsx
// example of SNS sent using node
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";
const client = new SNSClient(config);

const input = {
  TopicArn: "STRING_VALUE",
  TargetArn: "STRING_VALUE", // optional
  PhoneNumber: "STRING_VALUE", // optional
  Message: "STRING_VALUE",
  MessageAttributes: {
    "Version": {
      DataType: "Number",
      StringValue: "1"
    },
  },
  MessageDeduplicationId: "STRING_VALUE", // required for FIFO topics
  MessageGroupId: "STRING_VALUE", // required for FIFO topics
  Subject: "STRING_VALUE" // optional
};
const command = new PublishCommand(input);
const response = await client.send(command);
```

### Sources
* https://docs.aws.amazon.com/pt_br/sns/latest/dg/welcome.html
* https://docs.aws.amazon.com/pt_br/sns/latest/dg/sns-create-topic.html
* https://medium.com/@joudwawad/aws-sns-deep-dive-6cc9cefbb9bb
* https://aws.plainenglish.io/how-to-configure-an-event-driven-architecture-to-send-data-to-sns-3197acc88293
* https://medium.com/@joudwawad/notification-system-architecture-with-aws-968103c2c730