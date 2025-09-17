---
title: "End Event"
url: /refguide/end-event/
weight: 50
---

## Introduction

The **End Event** indicates where a workflow process ends.
When the workflow execution reaches this element, the process is completed and no further actions are taken.
Note that a workflow can have multiple end events, as shown in the example below:

{{< figure src="/attachments/refguide/modeling/application-logic/workflows/workflow-elements/end-event/multiple-end-events.png" alt="End Event Example" width="400" class="no-border" >}}

The example shows a decision activity, where the workflow can take two different paths based on a condition.
Each path leads to a different end event, either of which ends the process.
In addition, there's an interrupting boundary event on one of the user tasks, with its own end event.
In all of these cases, there's only one path that will be taken, and the process will end when it reaches the corresponding end event.

## Properties

The End Event properties does not have any properties.
