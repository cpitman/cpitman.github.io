---
published: false
layout: post
title: 'Waiting for Remote system in a Jenkins Pipeline, Now Easier'
date: 2017-03-31T00:00:00.000Z
categories: jenkins cicd
---
TLDR: I've release a new Jenkins plugin for integrating external systems into pipelines, the [Webhook Step Plugin](https://plugins.jenkins.io/webhook-step)!

Lets say we have a CICD pipeline that needs to kick off an external process (integration tests, stress tests, change control, etc) and wait for the process to finish before continuing? Busy looping is one way to do this, but this wastes resources and keeps a Jenkins Agent locked up for potentially long periods of time.

In a [previous post](/jenkins/cicd/2017/03/16/waiting-for-remote-systems-in-a-jenkins-pipeline.html) I walked through how to use the `input` step in a Jenkins Pipeline in order to wait for an external system to finish a task before continuing. While it works, it is harder than it should be:

* Since you are simulating posting a form, you need a separate service call to get a CSRF token
* Requires user authentication
* Requires the external system to post in a veyr specific format
* Can have race conditions where the external process finishes before the pipeline gets to the `input` step

Towards the end of the post, I asked "Is there a better way to do this?". Here is my answer: the [Webhook Step Plugin](https://plugins.jenkins.io/webhook-step) for Jenkins.

The Webhook Step plugin lets you create a one off webhook url that can be passed to an external system, and then wait for the external system to post results to Jenkins via the webhook. There are no requirements on the structure of the body, no need for a CSRF token, and authentication is handled by creating unique urls per use.

Using the pipeline step is about as easy as it gets:

1. Register a webhook
2. Trigger an external system
3. Wait for the external system to send results to Jenkins via the webhook

Here is an example that prints out the webhook url and waits for the user to post to it:

```
stage ("Long Running Stage") {
  hook = registerWebhook()

  echo "Waiting for POST to ${hook.getURL()}"

  data = waitForWebhook hook
  echo "Webhook called with data: ${data}"
}
```

When this job is executed, something like the following log is printed:

```
Waiting for POST to http://localhost:8080/webhook-step/bef13807-a161-4193-ab95-6cb974afc71d
```

To continue the pipeline, we can post to this url. To do this with curl, we would execute
`curl -X POST -d 'OK' http://localhost:8080/webhook-step/bef13807-a161-4193-ab95-6cb974afc71d`.
Looking back at the Jenkins Job, it would be complete and logged 
`Webhook called with data: OK`.


## Now What?

Webhook Step is meant to be a basic building block that allows integrations with all kinds of external systems, but that sounds awfully vague! I'll be posting soon an example of using Webhook Step with OpenShift (ie Kubernetes) Jobs so that tests and other actions can be sent to a compute cluster instead of tieing up Jenkins.

In the meantime, if you have any questions please reach out to me on [Twitter](http://twitter.com/home?status=@PitmanIO)!
