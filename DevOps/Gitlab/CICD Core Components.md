## Pipeline

Key points:
- **Definition**: YAML file named `.gitlab-ci.yml`.
- **Naming**: Use `workflow:name` to assign a custom name shown in the Pipelines tab.
- **Triggers**: Pipelines start on `push`, `merge_request`, `schedule`, or manual actions.
- **Conditions**: Use `rules` to control when pipelines run.
Example:
```yaml
workflow:
  name: My Awesome App Pipeline
  rules:
    - if: $CI_COMMIT_BRANCH == 'main'
```
## Stage
**Stages** define the sequence of execution in a pipeline. Jobs within the same stage run in parallel. Common stages include:

|Stage|Purpose|Example Commands|
|---|---|---|
|build|Compile or assemble code|`mvn package`, `docker build`|
|test|Execute automated tests|`npm test`, `pytest`|
|deploy|Release artifacts to an environment|`kubectl apply`, `aws s3 sync`|
|security|Run vulnerability scans|`trivy image`, `npm audit`
## Job

A **job** is the fundamental execution unit in a pipeline:
- **Execution**: Runs on a GitLab Runner (shared or self-hosted).
- **Runner selection**: Use `tags` to pick specific runners.
- **Stage assignment**: Each job belongs to a defined stage.
Example
```yaml
unit_test_job:
  stage: test
  tags:
    - saas-linux-small-amd64
```

> [!warning]
> Keep sensitive data out of `script` blocks. Use [CI/CD variables](https://docs.gitlab.com/ee/ci/variables/) for secrets like tokens or credentials.

## Script
The `script` section lists shell commands executed by the job:

- **Sequential execution**: Commands run in order.
- **Common use cases**: Install dependencies, build code, run tests, deploy artifacts, perform security scans.
- **Before/after hooks**:
    - `before_script`: Setup steps (e.g., install Node.js, configure DB).
    - `after_script`: Cleanup tasks (e.g., remove temp files).
```yaml
before_script:
  - echo "Setting up environment"
script:
  - npm install
  - npm test
after_script:
  - echo "Cleaning up"
```
## Example
Here’s a complete sample that defines two stages—`test` and `deploy`—and a workflow rule:

```yaml
workflow:
  name: My Awesome App Pipeline
  rules:
    - if: $CI_COMMIT_BRANCH == 'main'


stages:
  - test
  - deploy


unit_test_job:
  stage: test
  tags:
    - saas-linux-small-amd64
  before_script:
    - echo "Install NodeJS"
  script:
    - npm install
    - npm test
  after_script:
    - echo "Tests complete"


deploy_job:
  stage: deploy
  script:
    - echo "Deploying to production..."
```

In this configuration:

- The pipeline runs only on `main`.
- The **test** stage executes `unit_test_job` with setup, tests, and teardown.
- The **deploy** stage runs `deploy_job` after successful tests.

## Pipeline type
| Pipeline Type           | Description                                                                      |
| ----------------------- | -------------------------------------------------------------------------------- |
| Basic Pipeline          | Executes all jobs in each stage concurrently, then proceeds sequentially.        |
| DAG Pipeline            | Runs jobs based on explicit dependencies for maximum parallelism and efficiency. |
| Merge Request Pipeline  | Validates new changes in merge requests before merging.                          |
| Merged Results Pipeline | Tests the merged result of a MR to catch conflicts early.                        |
| Merge Train             | Queues MRs in sequence to streamline integration.                                |
| Parent-Child Pipelines  | Splits large pipelines into parent and child configs—ideal for monorepos.        |
| Multi-Project Pipelines | Coordinates pipelines across multiple repositories for cross-team workflows.     |