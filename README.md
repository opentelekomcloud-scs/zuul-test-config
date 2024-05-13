# ZUUL Config

## Zuul Tenant
```
- tenant:
    name: zuul-test
    report-build-page: true
    source:
      github:
        config-projects:
          - opentelekomcloud-scs/zuul-test-config
        untrusted-projects:
          - opentelekomcloud-scs/zuul-test
```
Assigns to a Tenant Name one or more git Repositories. 
- *untrusted:* Runs in a "secured" space with no posibillity to configure Zuul Core System. 
- *trusted or config:* The Zuul Config is defined here. Pipelines, nodepools, project-templates etc. are defined here. Any Component defined here runs in "trusted" space. So if you define a job here, it's trusted and has higher rights than if it is degined in a untrusted Repo.

## Zuul Pipeline
A Pipeline defines the triggers. With the Information in the Tenant Definition, a pipeline stats e.g. if a commit in a untrusted Repo takes place or a string "check_run" is written in the Comment of a pull_request.   
```
---
- pipeline:
    name: check
    description: |
      Newly opened pull requests enter this pipeline to receive an
      initial verification
    manager: independent
    trigger:
      github:
        - event: pull_request
          action:
            - opened
            - changed
            - reopened
        - event: pull_request
          action: comment
          comment: (?i)^\s*recheck\s*$
        - event: check_run
    start:
      github:
        check: 'in_progress'
        comment: false
    success:
      github:
        check: 'success'
    failure:
      github:
        check: 'failure'
```
Up to now, Zuul does not know what to execute... 

## Zuul Jobs
Jobs reference ansible playbooks which zull executes. 
```
---
- job:
    name: testbed-deploy-managerless
    nodeset: ubuntu-jammy
    pre-run: playbooks/managerless/pre.yml
    run: playbooks/managerless/deploy.yml
    post-run: playbooks/managerless/post.yml
    required-projects:
      - osism/terraform-base
    timeout: 10800
    vars:
      terraform_blueprint: testbed-managerless
      cloud_env: managerless-otc
    secrets:
      - name: secret
        secret: SECRET_TESTBED
    semaphores:
      - name: semaphore-testbed-managerless
```
There is no reference to the Pipeline, so it's a pure Definition. 

## Zuul Project
A Project brings togehter Pipelines (the trigger) and Jobs (the Task):
```
- project:
    merge-mode: squash-merge
    check:
      jobs:
        - flake8
        - ansible-lint
    post:
      jobs:
        - testbed-deploy-managerless
    periodic-hourly-otc:
      jobs:
        - ansible-lint
        - flake8
        - testbed-deploy-managerless
```
Here, the Project executes a job with a assigned ansible playbook 'testbed-deploy-managerless' when trigged by the post pipeline. The 'periodic-hourly' Pipeline executes the job 'testbed-deploy-managerless' too. But before, an 'ansible-lint' and 'flake8' is executed. Both aren't ansible playbooks. They are executed directly and should be confgured on the nodepool (binary) and within the untrused Repo (config).

## Zuul Secrets
To create a Zuul Secret, you have to install the Zuul-Client:
```
pip install zuul-client
```
Zuul Secrets are encrypted keys where the algorithm is dependend on the Zuul Installation, the Zuul Tenant, the Zuul Project and the file with the key to be encrypted:
```
zuul-client --zuul-url https://zuul.otc-scs.t-systems.net/ encrypt --tenant zuul-test --project opentelekomcloud-scs/zuul-test-config --infile ./otc_obs_ak 
writing RSA key

- secret:
    name: <name>
    data:
      <fieldname>: !encrypted/pkcs1-oaep
        - AqJisk0BlgujDQeyBYkb45lco2J4d8RQwDeNBi12345sa+ymAuot7onWQdtItEQY9Ap/5
...
          +onNDLm0vYD/3bJJKlLAM6p6Rde4ghwsk/N5EH7p0H2JH+osLG8MqwvKpYd4i4=

```








