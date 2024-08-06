---
layout: post
title:  Controlling access to macOS large runners for GitHub Actions
date:   2024-08-06 10:00:00
tags:   github github-actions
categories: eclipse
comments_id: 29
---

In 2023, GitHub introduced new powerful macOS runners for GitHub Actions. 
These [runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-larger-runners/running-jobs-on-larger-runners?platform=mac#available-macos-larger-runners) 
have a considerable higher amount of processors / memory and disk space allocated to them to speed up the execution of workflows.
This advantage comes at a cost though, as billing per minute of executed workflow time is considerably higher as compared to normal runners (see [billing for runners](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)), 
on top of usual minute multiplier for macOS runners (each minute of executed workflow time on a macOS runner counts as 10 minutes for billing purposes).

<br/>
In order to use such a macOS large runner, you can simply add a `runs-on: <runner-type>` to your job definition, e.g. using `macos-latest-large` as runner type:

{% highlight yaml %}
name: learn-github-actions-testing
on: [push]
jobs:
  build:
    runs-on: macos-latest-large
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: swift build
      - name: Run tests
        run: swift test
{% endhighlight %}

<br/>
Additionally, your organization needs to have a `GitHub Team` or `GitHub Enterprise Cloud` plan to be able to use such a macOS large runner, otherwise execution of
workflows using such a runner will fail to run. Once your organization is eligible to use large runners, you probably want to control the access to such runners for the repositories in your organization
to avoid surprises when you receive your next invoice. GitHub offers a convenient way to define [runner groups](https://docs.github.com/en/actions/using-github-hosted-runners/about-larger-runners/controlling-access-to-larger-runners) to define which repositories can access such large runners.

<br/>
Unfortunately, such runner groups can only be defined for `linux` and `windows` runners, there is simply no way to prevent that `macOS` large runners are being used by any of your repositories once they are configured in a workflow as described above. 
This poses a problem for non-profit organizations (like the [Eclipse Foundation](https://www.eclipse.org)) that host a lot of projects and their associated repositories on GitHub as it might result in higher than expected billing expenses as some projects try using such large runners
to speed up their workflows without realizing the consequences.

<br/>
While it is possible to monitor the incurred costs of using GitHub Action minutes, this is a tedious and manual task and requires communication with projects to change their workflows if occurrences have been identified.

<br/>
The idea was born to add some automation to prevent the execution of workflows on such `macOS` large runners unless the project / repository is entitled to use such a runner.

<br/>
After studying the available [GitHub Rest API](https://docs.github.com/en/rest?apiVersion=2022-11-28) and preliminary testing, we figured out the following logic reliably prevents the execution of workflows on large runners:

- listen to [workflow_job events](https://docs.github.com/en/webhooks/webhook-events-and-payloads?actionType=queued#workflow_job) with action `queued`
- check whether the included `workflow_job` object has `labels` that indicate that the job is supposed to run on a macOS large runner
- if the above evaluates to true and the repository is not eligible to use such a runner, [cancel the workflow_run](https://docs.github.com/en/rest/actions/workflow-runs?apiVersion=2022-11-28#cancel-a-workflow-run)

<br/>
To receive the necessary webhook events from GitHub in case a workflow is being scheduled to run you have to set up an organization or repository webhook, listen for the events and apply the logic. 

<br/>
At the [Eclipse Foundation](https://www.eclipse.org) we are operating an open-source project called [Otterdog](https://github.com/eclipse-csi/otterdog) in order configure our numerous organizations and repositories hosted on GitHub at scale.
This tool is effectively a GitHub App and is installed for all our projects / organizations on GitHub and already can listen to various events sent from GitHub. So naturally we added the above logic to this tool and allowed to define 
which organizations are allowed to use such large runners via a configuration file (see [this](https://github.com/eclipse-tractusx/.eclipsefdn/blob/main/otterdog/policies/macos_large_runners.yml) example).

<br/>
This allows us to control the use of macOS large runners which unfortunately is not yet possible through any of the administration consoles at GitHub. 
On the other hand, our implemented workaround showcases the power of GitHub Apps how you can utilize them to adjust your GitHub experience to your organizational needs.

<br/>
Feel free to leave comments on other useful things that you would like to see in the near future.