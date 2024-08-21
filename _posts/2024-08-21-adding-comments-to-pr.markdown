---
layout: post
title:  Automatically adding comments to GitHub PRs, safely and reliably
date:   2024-08-21 18:00:00
tags:   github github-actions
categories: eclipse
comments_id: 30
---

Using GitHub Actions offers a lot of possibilities to automate your development processes. Imagine you 
would like to get a coverage report for any PR that gets created for your repository and add a coverage summary directly
to the PR as a comments to make allow reviewers to immediately see if this PR meets the requirements wrt code coverage.

There are a lot of convenient actions available in the [GitHub Marketplace](https://github.com/marketplace?type=actions) 
that perform some action and add a comment to the PR. That usually works perfectly unless you come across the notorious error:

````{verbatim}
HttpError: Resource not accessible by integration
````

and your perfectly working workflow failed to add a comment to the PR that you just created. After some initial investigation you find out
that it is due to a PR created from a fork. Further digging reveals that GitHub treats PRs from forks differently for [security reasons](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/), namely
that in such case any workflow triggered by a ```pull_request``` trigger will not have access to any [secrets](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#workflows-in-forked-repositories) 
or [write tokens](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#permissions-for-the-github_token).

### Trying to be smart

GitHub takes "secure by default" seriously, so you don't easily get **pwned** by a malicious actor, however there are still ways to shoot yourself in the foot.
There is an additional workflow trigger [pull_request_target](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request_target) 
that will run a workflow from a PR in the context of the target repository, and as a consequence, have access to secrets and write tokens.

There are cases where doing something like that is acceptable and can be considered safe (see [Keeping your GitHub Actions safe](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)):

> Generally speaking, when the PR contents are treated as passive data, i.e. not in a position of influence over the build/testing process, it is safe. 

A very nice writeup about utilizing the ```pull_request_target``` trigger with some additional user permission checks can be found in this [blog entry](https://michaelheap.com/access-secrets-from-forks/). 
However, the downside of that approach is that you need to think hard about the attack surface of your workflows to prevent any unexpected side effects, something we ultimately want to avoid.

### Git (sic!) better

Some investigation of available [workflow triggers](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows) reveals the ```workflow_run```
trigger, that allows to react once a workflow has completed, and workflows triggered by that run in the context of the repository itself. That would allow us to do the following:

- run a workflow with ```pull_request``` trigger to perform some action, e.g. calculate the code coverage of the code in a PR
- store the result of this workflow as an artifact attached to the workflow run
- trigger another workflow upon completion of the first workflow using the ```workflow_run``` trigger with action ```completed```
- download the artifact attached to the completed workflow run
- add a comment to the associated PR based on the artifact data

We have implemented this in a real-world example for the [up-java repo](https://github.com/eclipse-uprotocol/up-java/) of the Eclipse uProtocol project.

### Gen(-AI+Mundane)

The [first workflow](https://github.com/eclipse-uprotocol/up-java/blob/main/.github/workflows/coverage.yml) will generate the relevant data for the PR,
and upload the generated data as an artifact. See the snippet below for the relevant steps:

{% highlight yaml %}
{% raw %}
...

- name: Generate coverage comment
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const COVERAGE_PERCENTAGE = `${{ env.COVERAGE_PERCENTAGE }}`;
      const COVERAGE_REPORT_PATH = `https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}/`;
        
      fs.mkdirSync('./pr-comment', { recursive: true });
        
      var pr_number = `${{ github.event.number }}`;
      var body = `
        Code coverage report is ready! :chart_with_upwards_trend:
        
        - **Code Coverage Percentage:** ${COVERAGE_PERCENTAGE}%
        - **Code Coverage Report:** [View Coverage Report](${COVERAGE_REPORT_PATH})
      `;
        
      fs.writeFileSync('./pr-comment/pr-number.txt', pr_number);
      fs.writeFileSync('./pr-comment/body.txt', body);

- uses: actions/upload-artifact@5d5d22a31266ced268874388b861e4b58bb5c2f3 # v4.3.1
  with:
    name: pr-comment
    path: pr-comment/
{% endraw %}
{% endhighlight %}

The comment that shall be added to the PR is stored in a file **body.txt** while the associated PR number is stored in a 
file **pr-number.txt** to know to which PR that comment should be added in the second workflow.

### Any comment?

Now that we have a workflow to calculate the code coverage of the PR and generate a comment that we would like to add to it,
we just need a [workflow](https://github.com/eclipse-uprotocol/up-java/blob/main/.github/workflows/coverage-comment-pr.yml) that gets triggered
when the first workflow is completed:

{% highlight yaml %}
{% raw %}
...

on:
  workflow_run:
    workflows: ["Java Test and Coverage"]
    types:
      - completed

jobs:
  add-coverage-comment:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    if: >
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.conclusion == 'success'
    steps:
      - name: 'Download artifact'
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        with:
          script: |
            var artifacts = await github.rest.actions.listWorkflowRunArtifacts({
               owner: context.repo.owner,
               repo: context.repo.repo,
               run_id: ${{github.event.workflow_run.id }},
            });
            var matchArtifact = artifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "pr-comment"
            })[0];
            var download = await github.rest.actions.downloadArtifact({
               owner: context.repo.owner,
               repo: context.repo.repo,
               artifact_id: matchArtifact.id,
               archive_format: 'zip',
            });
            var fs = require('fs');
            fs.writeFileSync('${{github.workspace}}/pr-comment.zip', Buffer.from(download.data));
      - run: unzip pr-comment.zip
      - name: 'Comment on PR'
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            var fs = require('fs');

            const issue_number = Number(fs.readFileSync('./pr-number.txt'));
            const body = fs.readFileSync('./body.txt', { encoding: 'utf8', flag: 'r' });

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue_number,
              body: body
            });
{% endraw %}
{% endhighlight %}

The workflow will download the artifact attached to the workflow run that triggered it and add a comment to the associated PR with the attached content.

> **_NOTE_**: There are pre-made [3rd party actions](https://github.com/marketplace?query=download+artifact+from+workflow+run) to 
download artifacts from previous workflow runs that could be used to further simplify such a workflow.

### Closing words

[GitHub.com](https://github.com) is a very powerful platform to host open-source projects, but you need to be aware that its open and collaborative model
also attracts malicious actors to attack or infiltrate (popular) repositories. Keep your repositories and workflows safe by following secure development best practices.

More information can be found in the [Security Handbook @ Eclipse Foundation](https://eclipse-csi.github.io/security-handbook/).