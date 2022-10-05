Here, you'll learn about GitHub Actions and workflows for CI. 

You'll learn to:

- Create a workflow from a template
- Understand the GitHub Actions logs
- Test against multiple targets
- Separate build and test jobs
- Save and access build artifacts
- Automate labeling a PR on review

## Create a workflow from a template

![Image](https://user-images.githubusercontent.com/6351798/193975447-ec70bb28-eb29-438c-8ed5-1e2f288954e7.png)

Building something from scratch can be a good learning experience, but if you've gone through the process before it can be time consuming and frustrating to repeat the same steps over and over. The same can be said for Actions workflows. Instead of creating a workflow from scratch, you can create a workflow using a pre-existing template. These templates have some common configurations, jobs, and even steps already defined for the particular type of automation you're implementing and type of language your working in. If you're not familiar with the basics of a GitHub Actions workflow, check out the [Automate development tasks by using GitHub Actions](/learn/modules/github-actions-automate-tasks/) module.

To see how you can create a new workflow from a template, you can navigate to the Code tab of your repository and select the **Actions** tab to create a new workflow. You'll then see a list of existing templates separated into categories for automation, continuous integration, deployment, security, and pages. Once a workflow is selected, you can then modify exisiting template. 

For an example, below are the contents from a existing Node.js template workflow. This template workflow will do a clean installation of node dependencies, cache/restore them, build the source code, and run tests across different versions of node. 

```yml
name: Node.js CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:

    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [14.x, 16.x, 18.x]
        # See supported Node.js release schedule at https://nodejs.org/en/about/releases/

    steps:
    - uses: actions/checkout@v3
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    - run: npm ci
    - run: npm run build --if-present
    - run: npm test
```

Notice the ```on:``` attribute. This workflow is triggered on a push to the repository, as well as when a pull request is made against the main branch.

There is one ```job``` in this workflow. Let's review what it does.

The ```runs-on:``` attribute specifies that, for the operating system, the workflow runs on ```ubuntu-latest```. The ```node-version:``` attribute specifies that there will be three builds, Node version 14.x, 16.x, and 18.x. We'll describe the ```matrix``` portion in depth later, when we customize the workflow.

The ```steps``` in the job use the GitHub Actions [actions/checkout@v3](https://github.com/actions/checkout?azure-portal=true) action to get the code from your repository into the VM, and the [actions/setup-node@v3error](https://github.com/actions/setup-node?azure-portal=true) action to set up the right version of Node.js. We specify that we're going to test two versions of Node.js with the ```${{ matrix.node-version }}``` attribute. This attribute points to the matrix we defined at the top of the file.

The last part of this step executes commands used by Node.js projects. The ```npm ci``` command installs dependencies from the *package-lock.json* file, ```npm run build --if-present``` runs a build script if it exists, and ```npm test``` runs the testing framework. Notice that this template includes both the build and test steps in the same job.

To learn more about npm, check out the npm documentation:

- [npm install](https://docs.npmjs.com/cli/install?azure-portal=true) 
- [npm run](https://docs.npmjs.com/cli/run-script?azure-portal=true) 
- [npm test](https://docs.npmjs.com/cli/test.html?azure-portal=true)

## Action Logs for the build

When a workflow runs, it produces a log that includes the details of what happened and any errors or test failures.
If there is an error or if a test has failed, you see a red X rather than a green check mark ✔️ in the logs. You can examine the details of the error or failure to investigate what went wrong.

![Image](https://user-images.githubusercontent.com/6351798/193976924-e3591837-c0b3-46f4-ac05-2beae5e1d3e6.png)

:::image type="content" source="../media/2-log-details.png" alt-text=" GitHub Actions log with details on a failed test." border="true":::

In the exercise, you identify failed tests by examining the details in the logs. You can access the logs from the **Actions** tab.

## Customize workflow templates

Recall that, at the beginning of this module, we described a scenario where you need to set up CI for your team. The Node.js template is a great start, but you want to customize it to better suit your own team's requirements. You want to target different versions of Node and different operating systems. You'll probably also want to separate the build and test steps into separate jobs.

Let's take a look at how you customize a workflow.

```yml
strategy:
matrix:
    os: [ubuntu-lastest, windows-2016]
    node-version: [14.x, 16.x, 18.x]
```

Here, we configured a [build matrix](https://docs.github.com/enterprise-server@3.1/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix) for testing across multiple operating systems and language versions. This matrix will produce four builds, one for each operating system paired with each version of Node.

Four builds, along with all their tests, will produce quite a bit of log information. It might be difficult to sort through it all. In the sample below, we show you how to move the test step to a dedicated test job. This job tests against multiple targets. Making the build and test steps separate will make it easier to understand the log.

```yml
test:
  runs-on: ${{ matrix.os }}
  strategy:
    matrix:
      os: [ubuntu-lastest, windows-2016]
      node-version: [14.x, 16.x, 18.x]
  steps:
  - uses: actions/checkout@v1
  - name: Use Node.js ${{ matrix.node-version }}
    uses: actions/setup-node@v1
    with:
      node-version: ${{ matrix.node-version }}
  - name: npm install, and test
    run: |
      npm install
      npm test
    env:
      CI: true
```

## What are artifacts?

When a workflow produces something other than a log entry, it's called an *artifact*. For example, the Node.js build will produce a Docker container that can be deployed. This artifact, the container, can be uploaded to storage by using the action [actions/upload-artifact](https://github.com/actions/upload-artifact?azure-portal=true) and downloaded from storage by using the action [actions/download-artifact](https://github.com/actions/download-artifact?azure-portal=true).

Storing an artifact helps to preserve it between jobs. Each job uses a fresh instance of a VM, so you can't reuse the artifact by saving it on the VM. If you need your artifact in a different job, you can upload the artifact to storage in one job and download it for the other job.

## Artifact storage

Artifacts are stored in storage space on GitHub. The space is free for public repositories and some amount is free for private repositories, depending on the account. GitHub stores your artifacts for 90 days.

In the following workflow snippet, notice that in the ```actions/upload-artifact@main``` action there is a ```path:``` attribute. This is the path to store the artifact. Here, we specify *public/* to upload everything to a directory. If it was just a file that we wanted to upload, we could use something like *public/mytext.txt*.

```yml
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: npm install and build webpack
        run: |
          npm install
          npm run build
      - uses: actions/upload-artifact@main
        with:
          name: webpack artifacts
          path: public/
```

To download the artifact for testing, the build must have completed successfully and uploaded the artifact. In the following code, we specify that the test job depends on the build job.

```yml
test:
    needs: build
    runs-on: ubuntu-latest
```

In the following workflow snippet, we download the artifact. Now the test job can use the artifact for testing.

```yml
steps:
    - uses: actions/checkout@v1
    - uses: actions/download-artifact@main
      with:
        name: webpack artifacts
        path: public
```

For more information about using artifacts in workflows see [Persisting workflow data using artifacts](https://help.github.com/actions/configuring-and-managing-workflows/persisting-workflow-data-using-artifacts?azure-portal=true) in the GitHub documentation.

## Automate reviews in GitHub using workflows

So far, we've described starting the workflow with GitHub events such as *push* or *pull-request*. We could also run a workflow on a schedule or on some event outside of GitHub.

Sometimes, we want to run the workflow after something a human needs to do. For example, we might only want to run a workflow after a reviewer has approved the pull request. For this scenario, we can trigger on ```pull-request-review```.

Another action we could take is to add a label to the pull request. In this case, we use the [pullreminders/label-when-approved-action](https://github.com/pullreminders/label-when-approved-action?azure-portal=true) action.

```yml
    steps:
     - name: Label when approved
       uses: pullreminders/label-when-approved-action@main
       env:
         APPROVALS: "1"
         GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
         ADD_LABEL: "approved"
```

Notice the block called ```env:```. This is where you set the environment variables for this action. For example, you can set the number of approvers needed. Here, it's one. The ```GITHUB_TOKEN``` variable is required because the action must make changes to your repository by adding a label. Finally, you supply the name of the label to add.

Adding a label could be an event that starts another workflow, such as a merge. We'll cover this in the next module on continuous delivery with GitHub Actions.
