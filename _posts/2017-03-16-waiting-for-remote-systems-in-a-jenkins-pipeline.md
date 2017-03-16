---
published: true
title: Waiting for Remote Systems in a Jenkins Pipeline
categories: jenkins cicd
---
When putting together a CICD pipeline, I often want to add stages that depend on a remote system executing a task that could potentially take a long time to complete. Or I want to integrate with a web based system that gives a callback when a task is complete (welcome to the future). 

The remote system could be doing lots of things:

1. Triggering and waiting for test results
2. Requesting a Code Review in a Code Review Tool and waiting for approval/rejection
3. Triggering a business process for change control and waiting for the result
4. Posting to a chat system and waiting for someone to type a command to continue

For example, one of the larger and more business critical systems I worked on had a set of integration tests that had a specialized scheduler that would execute tests across a cluster, and take ~8 hours to complete a full run. How can we integrate these long running tests into a Jenkins pipeline? 

Importantly, we don't want to tie up a Jenkins Agent for 8 hours doing nothing. It would be better if our external system could notify Jenkins when it is done, provide any results or other data, and then continue with the pipeline.

One Solution
------------

We can actually do this using the basic steps included in the Jenkins Pipeline plugin. The `input` step is usually used to block a pipeline while waiting for a user to complete some action, but by using Jenkins' REST API we can make an external system respond to an input.

Our pipeline needs to do the following:

1. Call our external system, passing a callback url
3. Call the `input` step and wait for the external system to trigger it

![Process Diagram]({{site.baseurl}}/images/pipeline_wait_for_webhook.png)

To do this, add something like the following to your Jenkinsfile:

```
stage("Wait for Remote System") {

  // Call a remote system to start execution, passing a callback url
  sh "curl -X POST -H 'Content-Type: application/json' -d '{\"callback\":\"${env.BUILD_URL}input/Async-input/proceedEmpty\"}' http://httpbin.org/post"

  // Block and wait for the remote system to callback
  input id: 'Async-input', message: 'Waiting for remote system'
}
```

Notice that we pass an `id` to the input step and that we use the exact same name as part of the callback url. If `id` isn't set Jenkins will generate a guid that can change when editing the step, so setting it makes the url predictable and stable.

Once we run this job, it will block once it gets to the input step. In order for our external system to respond, it will first need to get a CSRF token from Jenkins (Jenkins calls this a "crumb"). Once we have a crumb, we can then make a post to the callback url.

For testing, run the following commands to simulate this on the (bash!) command line:

```
CRUMB=`curl --user username:password 'http://jenkins-url/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)'`
curl -X POST -H "$CRUMB" --user username:password http://jenkins-url/job/async-demo/1/input/Async-input/proceedEmpty
```

Once the above script is executed, the pipeline will continue to execute!

Passing Data
------------

So far we have only resumed a paused job. What if the external system also produces data that needs to be pulled in to our pipeline, like test results?

The `input` step also supports parameters, and we can post those parameters through the API as well. The easiest way to figure out the right parameters is to use your browser's developer tools and watch how the web console responds when an input form is submitted.

Say we wanted to upload a file. First, the input step should change:

```
input id: 'Async-input', message: 'Waiting for remote system', parameters: [file(description: 'Performance Test Results', name: 'results/performance.out')]
```

Then, when the remote system responds to the callback it should submit a file as data. Using curl, it would look like this:

```
curl -X POST --form "$CRUMB" --user username:password \
  --form file0=@PATH_TO_RESULTS_FILE \
  --form json='{"parameter": [{"name":"results/performance.out", "file":"file0"}]}' \
  --form proceed=Proceed \
  http://jenkins-url/job/async-demo/1/input/Async-input/submit
```

The `@` symbol is important! It tells curl that what follows is a filename so that it will read and post the data from the file.

Another Solution?
-----------------

Using the input plugin has a couple warts that make it more difficult to work with:

1. It requires authorization by the remote system using user credentials
2. It requires the remote system to get and use a CSRF token
3. Especially when posting data, the format of the request has to be the very specific format supported by the Jenkins `input` step
4. There is a small window for timing issues, where the remote system could finish and respond _really_ fast, before Jenkins gets to the `input` step

All of these mean that we have to adjust the remote system to work with Jenkins, and cannot just use a standard webhook feature. Ideally, another plugin would be written that generates unique callback urls for each execution, getting rid of the need for CSRF and authorization. It would also accept any data, and let the pipeline script figure out what to do with it.

Such a plugin would let us rewrite the pipeline like this:

```
stage("Wait for Remote System") {
  callback_url = registerWebhook()

  // Call a remote system to start execution, passing a callback url
  sh "curl -X POST -H 'Content-Type: application/json' -d '{\"callback\":\"${callback_url}"}' http://httpbin.org/post"

  // Block and wait for the remote system to callback
  waitForWebhook callback_url
}
```

Testing this with curl we would execute `curl -X POST $callback_url`. Since we don't have to muck with authorization or CSRF, that's all that would be required. Any system that supports callbacks would then be easy to integrate with!
