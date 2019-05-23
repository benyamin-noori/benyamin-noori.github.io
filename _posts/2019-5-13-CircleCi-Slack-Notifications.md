---
layout: post
title: Getting Slack Notifications For CircleCi 2.0 Workflows
---

[CircleCi](https://circleci.com/) 2.0 has introduced workflows. A workflow is a set of declarations that define a number of jobs, their run order and helps you orchestrate and troubleshoot those jobs. It has a lot of benefits compared to CircleCi 1.0 that doesn't support workflows. However, our team switched to CircleCi 2.0, getting notifications on our continuous integration process became a problem. Our team usuallys gets their notifications through Slack. We have a channel and all our CI/CD processes/pipelines report to that slack channel.

## CircleCi Slack App
CircleCi has its own Slack integration. Great, we're done here. Right? Well... that really depends on what you're looking for. 

## Slack Notifications in CircleCi 1.0
If you want to get slack notifications for CircleCi 1.0, you can do one of two things: 

### CircleCi Slack App 
This integration allows you to get notifications for your CI jobs on Slack, but it has an important drawback: You can't limit your notifications to a specific branch (e.g. "master"). It will notify you of all jobs, including CI jobs for pull requests, which isn't great. Our team wanted to be notified of the build status, only on the master branch. Poeple were working on their topic branches all day, every day. If we were to get a notification for every CI job running on every commit on every branch, that would be a mess.

In order to set this up, we just have to add the CircleCi integration to our Slack workspace.

### Slack Webhook
[Slack Webhooks](https://api.slack.com/incoming-webhooks) are a good way to integrate any custom service you want into your Slack environment, and you can use them with CircleCi 1.0 to get notifications on your jobs. However, there's no very easy way to do this, you have to implement it yourself. You have to make the Webhook URL available to your CI job (e.g. through an environment variable), and make a request to that URL with whatever message you want to send. If you want to limit your notifications to certain branches, you can do that, too. This is an example of a CircleCi config file that allows us to run certains steps only on the master branch:

```
version: 2
jobs:
  build:
    working_directory: ~/code
    docker:
    - image: circleci/node:8.10
    steps:
      - checkout
      - run: npm install
      - run: npm test
      - run: if [ "$CIRCLE_BRANCH" == "master" ]; then npm publish; fi
```

Obviously, this approach comes with a lot of flexibility, but it's more effort than our other options! So, what does all of this look like in CircleCi 2.0? 

## Slack Notifications in CircleCi 2.0
Did CircleCi 2.0 workflows solve any of those problems? Yes.

In CircleCi 2.0, you can add certain filters and apply those filters to any of the steps in any of your jobs. One of those filters can be to run something only if the current branch is named "master". That way, I can get notifications only for the master branch, YAY! In the following example, we define a workflow that runs the deploy job only when the current branch is master. That way, only commits to master will get deployed.

```
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#
version: 2

defaults: &defaults
  docker:
    - image: circleci/node:8.10

references:
    run_on_master: &run_on_master
      filters:
        branches:
          only: master
jobs:
  build:
    <<: *defaults

    steps:
      - checkout
      - run: npm install
      - persist_to_workspace:
          root: ../
          paths: ./*

  test:
    <<: *defaults
    steps:
      - attach_workspace:
          at: ~/
      # run tests!
      - run: npm test

  deploy:
    <<: *defaults
    steps:
      - attach_workspace:
          at: ~/
      - run: npm publish

workflows:
  version: 2
  default:
    jobs:
      - build:
          context: ci-read
      - deploy:
          <<: *run_on_master
          context: ci-write
```

Also, with CircleCi 2.0, you share environment variables among all projects in your org. So if you're storing an API key, any credentials or constant values in environment variables in your org, that's great too.

So workflows are great, but the way notifications are set up with Workflows is annoying. You can enable Slack notifications in your project settings on the CircleCi website. CircleCi has setup workflow notifications on a per-job basis, not on a per-workflow basis. What does that mean? Let's assume I have defined a workflow that consists of 3 jobs: Build, Test, and Deploy (like the example above).

Using the CircleCi Slack integration, for every CI workflow that runs, I might get three notifications! That's not great. If you have a lot of branches with people actively working on them, it'll end up spamming your channel all the time. 

Even if you limit your notifications to a certain branch (e.g. master), it will still post three notifications per build on master. As stated before, our team has a number of repos that use CircleCi and we would like to get notifications for all those repos all in one channel. That means even if we limit our notifications to the master branch we might still end up getting a lot of notifications on Slack, 3 per each CI run. Not great.

### Enter Slack Orbs
CircleCi has been nice enough to provide a way for us to _really_ customize our Slack integration. That means customizing everything from the message it sends out and the information, links, actions and statuses included. You can get an idea of what Orbs are and how to use them [here](https://github.com/CircleCI-Public/slack-orb).

So let's review what we want to accomplish: 

* Get notifications for CI jobs on Slack: Slack orbs needs us to define a `SLACK_WEBHOOK` environment variable. The value of this variable is the webhook url provided to us by Slack. All we need to do is go to Slack and generate a Slack webhook url for the channel we want to post our notifications to, and then add an environment variable to our project (in CircleCi project settings).
* Get notifications only for the "master" branch: We can limit notifications to the master branch in CircleCi 2.0! Just like the example above.
* Get only one notification per each workflow: This is the part that's a little bit tricky. So, an important thing to realize, is that in every workflow, every job runs if and only if the jobs before it finish successfully. So to return to our previous example of Build, Test, Deploy, the Test job only runs if the Build job exits with code 0. The same is true for the Deploy job. So, how can we take advantage of this? When we set up notifications to the Slack webhook url using Slack Orbs, we can tell it to notify us when either success, failure, or both happens. So if I setup my notifications the following way: 

    * Build: Notify Slack only on failure. 
    * Test: Notify Slack only on failure. 
    * Deploy: Notify Slack either way.

This way, if the build job fails, we get a notification on Slack and realize something's gone wrong. The same is true for the Test and Deploy jobs. However, if all goes well, we only get one notification, which will be a success notification for the deploy job, letting us know that all jobs in the workflow have succeeded. Here's an example CircleCi config file that demonstrates how to do all of this:


```
# Javascript Node CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-javascript/ for more details
#
version: 2.1

orbs:
  slack: circleci/slack@2.2.0

defaults: &defaults
  docker:
    - image: circleci/node:8.10

references:
    notifications: &notifications
      only_for_branch: "master"
      include_project_field: true
      include_job_number_field: true
      include_visit_job_action: true

    run_on_master: &run_on_master
      filters:
        branches:
          only: master

jobs:

  build:
    <<: *defaults

    steps:
      - checkout
      - run: npm install
      - persist_to_workspace:
          root: ../
          paths: ./*
      - slack/status:
          <<: *notifications
          fail_only: "true"

  test:
    <<: *defaults
    steps:
      - attach_workspace:
          at: ~/
      - run: npm test
      - slack/status:
          <<: *notifications
          fail_only: "true"

  deploy:
    <<: *defaults
    steps:
      - attach_workspace:
          at: ~/
      - run: npm publish
      - slack/status:
          <<: *notifications

workflows:
  version: 2
  default:
    jobs:
      - build:
          context: ci-read
          filters:
            tags:
              only: /.*/
      - test:
          requires: [build]
          filters:
            tags:
              only: /.*/
      - deploy:
          <<: *run_on_master
          context: ci-write
          requires: [test]
```