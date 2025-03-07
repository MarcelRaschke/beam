<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
      http://www.apache.org/licenses/LICENSE-2.0
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

# Guidelines for Adding or Modifying Workflows

On top of normal Actions workflow steps, all new CI workflows (excluding release workflows or other workflow automation) should have the following:

1) Name and phrase set via matrix for re-run to work (See below)
2) A set of specific triggers
3) An explicit checkout step
4) A set of GitHub token permissions
5) Concurrency Groups
6) Comment Triggering Support

Each of these is described in more detail below.
## Name and Phrase
Due to specifics on how the comment triggered rerun is handled it is required that all jobs have name and phrase set via matrix elements. See the following example:
```
jobs:
  beam_job_name:
    name: ${{ matrix.job_name }} (${{ matrix.job_phrase }})
    strategy:
      matrix:
        job_name: [beam_job_name]
        job_phrase: [Run Job Phrase]
    if: |
      github.event_name == 'push' ||
      github.event_name == 'pull_request_target' ||
      github.event_name == 'schedule' ||
      github.event_name == 'workflow_dispatch' ||
      github.event.comment.body == 'Run Job Phrase'
    steps:
      - uses: actions/checkout@v3
      - name: Setup repository
        uses: ./.github/actions/setup-action
        with:
          comment_phrase: ${{ matrix.job_phrase }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          github_job: ${{ matrix.job_name }} (${{ matrix.job_phrase }})

```
And in case when the workflow already utilizes matrix do the following:
```
jobs:
  beam_job_with_matrix:
    name: ${{ matrix.job_name }} (${{ matrix.job_phrase }} ${{ matrix.python_version }})
    runs-on: [self-hosted, ubuntu-20.04, main]
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        job_name: ["beam_job_with_matrix"]
        job_phrase: ["Run Job With Matrix"]
        python_version: ['3.8','3.9','3.10','3.11']
    if: |
      github.event_name == 'push' ||
      github.event_name == 'pull_request_target' ||
      github.event_name == 'schedule' ||
      startsWith(github.event.comment.body, 'Run Job With Matrix')
    steps:
      - uses: actions/checkout@v3
      - name: Setup repository
        uses: ./.github/actions/setup-action
        with:
          comment_phrase: ${{matrix.job_phrase}}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          github_job: ${{ matrix.job_name }} (${{ matrix.job_phrase }} ${{ matrix.python_version }})
```
Notice how the matrix element of `python_version` is appended to the name.

## Workflow triggers

GitHub allows workflows to define a set of triggers that dictate when a job should be run. For more info, see [documentation here](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows).

For the purposes of Beam, each CI workflow should define the following triggers:

1) A `push` trigger
2) A `pull_request_target` trigger
3) An issue_comment trigger (for issue created). This is needed for comment triggering support (see section below).
4) A scheduled trigger
5) A workflow_dispatch trigger

The `push`/`pull_request_target` triggers should only run when appropriate paths are modified. See https://github.com/apache/beam/blob/master/.github/workflows/beam_PreCommit_Go.yml#L4 for an example (you can copy and paste this into your workflow, you just need to change the paths).

## Checkout step

Because we use the `pull_request_target` trigger instead of `pull_request`, we need an explicit checkout of the correct commit. This can be done as a step that uses the `setup-action` action in your workflow. See https://github.com/apache/beam/blob/0ee2dc73ec6f555a5bf1a643dffd37f4927be67e/.github/workflows/beam_PreCommit_Go.yml#L65-L70 for an example (you can copy and paste this into your workflow). Please make sure that you checkout the code before using the composite action.

## Token Permissions

You should explicitly define the GitHub Actions token scopes needed by your job. For most jobs, this will be `actions: write` (needed for comment triggering support) and `read` permissions for all other options. See https://github.com/apache/beam/blob/907c5110163b0efe52e9e12127fd013c7fc455d7/.github/workflows/beam_PreCommit_Go.yml#L16 for an example (you can copy and paste this into your workflow).

## Concurrency Groups

Concurrency groups are a way of making sure that no more than one Actions run is running per job/group at any given time (previous ones can be cancelled). To reduce resource usage, you should define the following concurrency group:

```
concurrency:
  group: '${{ github.workflow }} @ ${{ github.event.pull_request.head.label || github.sha || github.head_ref || github.ref }}-${{ github.event.sender.login }}-${{ github.event.schedule }}'
  cancel-in-progress: true
```

this defines the following groups:

1) Each workflow will represent a different set of groups
2) Within each workflow, each PR will represent a single group
3) Within each non-PR run for a workflow, each commit will represent a set of groups
4) Within each commit, each push event, schedule event, and manual run event will represent a set of groups

## Comment Triggering Support

In order to make it easier for non-committers to interact with workflows, workflows should implement comment triggering support. This requires 3 additional components beyond the ones mentioned above:

1) Each job should be gated on an if condition that maps to that workflow's comment trigger (example: https://github.com/apache/beam/blob/907c5110163b0efe52e9e12127fd013c7fc455d7/.github/workflows/beam_PreCommit_Go.yml#L38)
2) Each job should have the rerun action immediately after its checkout step. You can add a step that uses the `setup-action` action in your workflow, which encapsulates the checkout and rerun logic in one place. This should be gated on the comment trigger (example: https://github.com/apache/beam/blob/0ee2dc73ec6f555a5bf1a643dffd37f4927be67e/.github/workflows/beam_PreCommit_Go.yml#L65-L70)
3) Each job should have a descriptive name that includes the comment trigger (example: https://github.com/apache/beam/blob/ba8fc935222aeb070668fbafd588bc58e7a21289/.github/workflows/beam_PreCommit_CommunityMetrics.yml#L48)

# Testing new workflows or workflow updates

## Testing New Workflows

New workflows are not added to the [UI on the Actions tab](https://github.com/apache/beam/actions) until they are merged into the repo's main branch (master for Beam).
To test new workflows, we recommend the following pattern:

1) Fork the Beam repo
2) Add your proposed workflow to the main branch of your fork.
3) Run the workflow in the [Actions tab](https://github.com/apache/beam/actions) of your fork using the `Run workflow` button.

Note: most workflows use [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)
with the main and ubuntu labels to execute ([example](https://github.com/apache/beam/blob/5a54ee6ddd8cb8444c41802929a364fe2561001e/.github/workflows/beam_PostCommit_Go_Dataflow_ARM.yml#L41)).
If you are testing on a fork, you likely will not have self-hosted runners set up.
To work around this, you can start using hosted runners and then switch over when you're ready to create a PR.
You can do this by changing `runs-on: [self-hosted, ubuntu-20.04, main]` (self-hosted, use in your PR) to `runs-on: ubuntu-20.04` (GitHub hosted, use for local testing).

## Testing Workflow Updates

If you need to make more changes to the workflow yaml file after it has been added to the repo, you can develop normally on a branch (using your fork or the main Beam repo if you are a committer). If you are a non-committer, this flow has several caveats mentioned below.
To do this:

1) Make your change on a development branch.
2) Navigate to your workflow in the [Actions tab](https://github.com/apache/beam/actions). If your changes are on a fork, navigate to the fork's Actions tab instead. If you don't see the correct action, make sure that your fork's main branch is up to date with Beam's master branch.
3) Click run workflow. Before clicking submit, update to run on your branch.

Note: If you run a workflow from your fork of Beam, it will not have access to secrets stored in the Beam repository.
This will cause some things like authenticating to external services to fail, which may cause your workflow to fail.
If you run into this issue, you can either:
1) Ask for a committers help to add the workflow to a branch on the main apache/beam repo.
2) Upload secrets to your repo mirroring the secrets used in the main Beam repo.
3) Wait until the changes are merged into Beam to test (this should only be done rarely).

Additionally, as mentioned above your fork likely will not have self-hosted runners set up.
To work around this, you can start using hosted runners and then switch over when you're ready to create a PR.
You can do this by changing runs-on: [self-hosted, ubuntu-20.04, main] (self-hosted, use in your PR) to runs-on: ubuntu-20.04 (GitHub hosted, use for local testing).

# Workflows
Please note that jobs with matrix need to have matrix element in the comment. Example:
```Run Python PreCommit (3.8)```
| Workflow name | Matrix | Trigger Phrase | Cron Status |
|:-------------:|:------:|:--------------:|:-----------:|
| [ PostCommit Go Dataflow ARM](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Go_Dataflow_ARM.yml) | N/A |`Run Go PostCommit Dataflow ARM`| [![.github/workflows/beam_PostCommit_Go_Dataflow_ARM](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Go_Dataflow_ARM.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Go_Dataflow_ARM.yml) |
| [ PostCommit Java Examples Dataflow ARM ](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Java_Examples_Dataflow_ARM.yml) | N/A |`Run Java_Examples_Dataflow_ARM PostCommit`| [![PostCommit Java Examples Dataflow ARM](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Java_Examples_Dataflow_ARM.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PostCommit_Java_Examples_Dataflow_ARM.yml) |
| [ PreCommit Community Metrics ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_CommunityMetrics.yml) | N/A |`Run CommunityMetrics PreCommit`| [![.github/workflows/beam_PreCommit_CommunityMetrics.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_CommunityMetrics.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_CommunityMetrics.yml) |
| [ PreCommit Go ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Go.yml) | N/A |`Run Go PreCommit`| [![.github/workflows/beam_PreCommit_Go.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Go.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Go.yml) |
| [ PreCommit Java Amazon Web Services IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services_IO_Direct.yml) | N/A |`Run Java_Amazon-Web-Services_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Amazon-Web-Services_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services_IO_Direct.yml) |
| [ PreCommit Java Amazon Web Services2 IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services2_IO_Direct.yml) | N/A |`Run Java_Amazon-Web-Services2_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Amazon-Web-Services2_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services2_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amazon-Web-Services2_IO_Direct.yml) |
| [ PreCommit Java Amqp IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amqp_IO_Direct.yml) | N/A |`Run Java_Amqp_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Amqp_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amqp_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Amqp_IO_Direct.yml) |
| [ PreCommit Java Azure IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Azure_IO_Direct.yml) | N/A |`Run Java_Azure_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Azure_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Azure_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Azure_IO_Direct.yml) |
| [ PreCommit Java Cassandra IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cassandra_IO_Direct.yml) | N/A |`Run Java_Cassandra_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Cassandra_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cassandra_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cassandra_IO_Direct.yml) |
| [ PreCommit Java Cdap IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cdap_IO_Direct.yml) | N/A |`Run Java_Cdap_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Cdap_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cdap_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Cdap_IO_Direct.yml) |
| [ PreCommit Java Clickhouse IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Clickhouse_IO_Direct.yml) | N/A |`Run Java_Clickhouse_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Clickhouse_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Clickhouse_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Clickhouse_IO_Direct.yml) |
| [ PreCommit Java Csv IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Csv_IO_Direct.yml) | N/A |`Run Java_Csv_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Csv_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Csv_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Csv_IO_Direct.yml) |
| [ PreCommit Java Examples Dataflow ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow.yml) | N/A |`Run Java_Examples_Dataflow PreCommit`| [![.github/workflows/beam_PreCommit_Java_Examples_Dataflow.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow.yml) |
| [ PreCommit Java Flink Versions ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Flink_Versions.yml) | N/A |`Run Java_Flink_Versions PreCommit`| [![.github/workflows/beam_PreCommit_Java_Flink_Versions.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Flink_Versions.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Flink_Versions.yml) |
| [ PreCommit Java Examples Dataflow Java11 ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow_Java11.yml) | N/A | `Run Java_Examples_Dataflow_Java11 PreCommit` | [![.github/workflows/beam_PreCommit_Java_Examples_Dataflow_Java11.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow_Java11.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Examples_Dataflow_Java11.yml) |
| [ PreCommit Java HBase IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HBase_IO_Direct.yml) | N/A |`Run Java_HBase_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_HBase_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HBase_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HBase_IO_Direct.yml) |
| [ PreCommit Java HCatalog IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HCatalog_IO_Direct.yml) | N/A |`Run Java_HCatalog_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_HCatalog_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HCatalog_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_HCatalog_IO_Direct.yml) |
| [ PreCommit Java InfluxDb IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_InfluxDb_IO_Direct.yml) | N/A |`Run Java_InfluxDb_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_InfluxDb_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_InfluxDb_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_InfluxDb_IO_Direct.yml) |
| [ PreCommit Java Kudu IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Kudu_IO_Direct.yml) | N/A |`Run Java_Kudu_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Kudu_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Kudu_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Kudu_IO_Direct.yml) |
| [ PreCommit Java Redis IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Redis_IO_Direct.yml) | N/A |`Run Java_Redis_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Redis_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Redis_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Redis_IO_Direct.yml) |
| [ PreCommit Java Solr IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Solr_IO_Direct.yml) | N/A |`Run Java_Solr_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Solr_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Solr_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Solr_IO_Direct.yml) |
| [ PreCommit Java Spark3 Versions ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Spark3_Versions.yml) | N/A | `Run Java_Spark3_Versions PreCommit` | [![.github/workflows/beam_PreCommit_Java_Spark3_Versions.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Spark3_Versions.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Spark3_Versions.yml) |
| [ PreCommit Java Splunk IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Splunk_IO_Direct.yml) | N/A |`Run Java_Splunk_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Splunk_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Splunk_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Splunk_IO_Direct.yml) |
| [ PreCommit Java Thrift IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Thrift_IO_Direct.yml) | N/A |`Run Java_Thrift_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Thrift_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Thrift_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Thrift_IO_Direct.yml) |
| [ PreCommit Java Tika IO Direct ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Tika_IO_Direct.yml) | N/A |`Run Java_Tika_IO_Direct PreCommit`| [![.github/workflows/beam_PreCommit_Java_Tika_IO_Direct.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Tika_IO_Direct.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Java_Tika_IO_Direct.yml) |
| [ PreCommit Python ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python PreCommit (matrix_element)` | [![.github/workflows/beam_PreCommit_Python.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python.yml) |
| [ PreCommit Python Coverage ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Coverage.yml) | N/A | `Run Python_Coverage PreCommit`| [![.github/workflows/beam_PreCommit_Python_Coverage.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Coverage.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Coverage.yml) |
| [ PreCommit Python Dataframes ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Dataframes.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python_Dataframes PreCommit (matrix_element)`| [![.github/workflows/beam_PreCommit_Python_Dataframes.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Dataframes.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Dataframes.yml) |
| [ PreCommit Python Docker ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocker.yml) | ['3.8','3.9','3.10','3.11'] | `Run PythonDocker PreCommit (matrix_element)`| [![.github/workflows/beam_PreCommit_PythonDocker.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocker.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocker.yml) |
| [ PreCommit Python Docs ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocs.yml) | N/A | `Run PythonDocs PreCommit`| [![.github/workflows/beam_PreCommit_PythonDocs.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocs.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonDocs.yml) |
| [ PreCommit Python Examples ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Examples.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python_Examples PreCommit (matrix_element)` | [![.github/workflows/beam_PreCommit_Python_Examples.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Examples.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Examples.yml) |
| [ PreCommit Python Formatter ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonFormatter.yml) | N/A | `Run PythonFormatter PreCommit`| [![.github/workflows/beam_PreCommit_PythonFormatter.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonFormatter.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonFormatter.yml) |
| [ PreCommit Python Integration](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Integration.yml) | ['3.8','3.11'] | `Run Python_Integration PreCommit (matrix_element)` | [![.github/workflows/beam_PreCommit_Python_Integration.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Integration.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Integration.yml) |
| [ PreCommit Python Lint ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonLint.yml) | N/A | `Run PythonLint PreCommit` | [![.github/workflows/beam_PreCommit_PythonLint.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonLint.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_PythonLint.yml) |
| [ PreCommit Python Runners ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Runners.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python_Runners PreCommit (matrix_element)`| [![.github/workflows/beam_PreCommit_Python_Runners.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Runners.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Runners.yml) |
| [ PreCommit Python Transforms ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Transforms.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python_Transforms PreCommit (matrix_element)`| [![.github/workflows/beam_PreCommit_Python_Transforms.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Transforms.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Python_Transforms.yml) |
| [ PreCommit RAT ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_RAT.yml) | N/A | `Run RAT PreCommit` | [![.github/workflows/beam_PreCommit_RAT.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_RAT.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_RAT.yml) |
| [ PreCommit SQL Java11 ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java11.yml) | N/A |`Run SQL_Java11 PreCommit`| [![.github/workflows/beam_PreCommit_SQL_Java11.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java11.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java11.yml) |
| [ PreCommit SQL Java17 ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java17.yml) | N/A |`Run SQL_Java17 PreCommit`| [![.github/workflows/beam_PreCommit_SQL_Java17.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java17.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_SQL_Java17.yml) |
| [ PreCommit Typescript ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Typescript.yml) | N/A |`Run Typescript PreCommit`| [![.github/workflows/beam_PreCommit_Typescript.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Typescript.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Typescript.yml) |
| [ PreCommit Website ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website.yml) | N/A |`Run Website PreCommit`| [![.github/workflows/beam_PreCommit_Website.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website.yml) |
| [ PreCommit Website Stage GCS ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website_Stage_GCS.yml) | N/A |`Run Website_Stage_GCS PreCommit`| [![PreCommit Website Stage GCS](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website_Stage_GCS.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Website_Stage_GCS.yml) |
| [ PreCommit Whitespace ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Whitespace.yml) | N/A |`Run Whitespace PreCommit`| [![.github/workflows/beam_PreCommit_Whitespace.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Whitespace.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Whitespace.yml) |
| [ Python Validates Container Dataflow ARM ](https://github.com/apache/beam/actions/workflows/beam_Python_ValidatesContainer_Dataflow_ARM.yml) | ['3.8','3.9','3.10','3.11'] | `Run Python ValidatesContainer Dataflow ARM (matrix_element)`| [![.github/workflows/beam_Python_ValidatesContainer_Dataflow_ARM.yml](https://github.com/apache/beam/actions/workflows/beam_Python_ValidatesContainer_Dataflow_ARM.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_Python_ValidatesContainer_Dataflow_ARM.yml) |
| [ PreCommit GoPortable ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_GoPortable.yml) | N/A |`Run GoPortable PreCommit`| [![.github/workflows/beam_PreCommit_GoPortable.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_GoPortable.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_GoPortable.yml) |
| [ PreCommit Kotlin Examples ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Kotlin_Examples.yml) | N/A | `Run Kotlin_Examples PreCommit` | [![.github/workflows/beam_PreCommit_Kotlin_Examples.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Kotlin_Examples.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Kotlin_Examples.yml) |
| [ PreCommit Portable Python ](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Portable_Python.yml) | ['3.8','3.11'] | `Run Portable_Python PreCommit` | [![.github/workflows/beam_PreCommit_Portable_Python.yml](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Portable_Python.yml/badge.svg?event=schedule)](https://github.com/apache/beam/actions/workflows/beam_PreCommit_Portable_Python.yml) |
