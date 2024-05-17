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
zuul-client --zuul-url https://zuul.otc-scs.t-systems.net/ encrypt --tenant scs-stack --project github.com/opentelekomcloud-scs/zuul-test-config --infile ./otc_obs_ak 
writing RSA key

- secret:
    name: <name>
    data:
      <fieldname>: !encrypted/pkcs1-oaep
        - AqJisk0BlgujDQeyBYkb45lco2J4d8RQwDeNBi12345sa+ymAuot7onWQdtItEQY9Ap/5
...
          +onNDLm0vYD/3bJJKlLAM6p6Rde4ghwsk/N5EH7p0H2JH+osLG8MqwvKpYd4i4=

```
and Secret Key
```
zuul-client --zuul-url https://zuul.otc-scs.t-systems.net/ encrypt --tenant scs-stack --project opentelekomcloud-scs/zuul-test-config --infile ./otc_obs_sk 
writing RSA key

- secret:
    name: <name>
    data:
      <fieldname>: !encrypted/pkcs1-oaep
        - AqJisk0BlgujDQeyBYkb45lco2J4d8RQwDeNBi12345sa+ymAuot7onWQdtItEQY9Ap/5
...
          +onNDLm0vYD/3bJJKlLAM6p6Rde4ghwsk/N5EH7p0H2JH+osLG8MqwvKpYd4i4=
```

# ZUUL Development 
The ansible scripts executed by the zuul-executor are stored within the dir `/var/lib/zuul/builds/<hash>/<jobname>/project_0/<project>/playbooks`, eg.g. `/var/lib/zuul/builds/939540125f094e5db6a3a722b3ff895a/untrusted/project_0/github.com/opentelekomcloud-scs/zuul-test/playbooks` but the playbooks get deleted by the cleanup job 

The output of the ansible can be read in the logs (argocd) of the zuul-executor or with
```
kubectl logs -n zuul-ci -l app.kubernetes.io/component==zuul-executor  -c zuul -f
```
Before executing kubectl you have to setup a tunnel with make tunnel, make kube within the k8s_development repo.

# ZUUL Troubeshooting

## Login 
There are two prosibilities to logon:
   * argocd shell
   * make tunnel && make kube

Here the second option is described:

### Scheduler
```
 kubectl exec -it -n zuul-ci $(kubectl get pod -n zuul-ci -l app.kubernetes.io/name=zuul,app.kubernetes.io/component=zuul-scheduler -o jsonpath='{.items[0].metadata.name}') -c zuul -- bash
```
### Executor
``` 
kubectl exec -it -n zuul-ci $(kubectl get pod -n zuul-ci -l app.kubernetes.io/name=zuul,app.kubernetes.io/component=zuul-executor -o jsonpath='{.items[0].metadata.name}') -c zuul -- bash
```

## Check tenant
```
zuul-admin tenant-conf-check
```

## Create auth string
- [ZUUL Client Docu](https://zuul-ci.org/docs/zuul/latest/client.html)

In scheduler do
```
zuul-admin  create-auth-token --auth-config zuul_operator --user alice --tenant scs-stack --expires-in 18000
```
Copy the resulting string into `~/.zuul.conf`
```
[scs]
url=https://zuul.otc-scs.t-systems.net/
auth_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwOi8vbWFuYWdlc2Yuc2ZyZG90ZXN0aW5zdGFuY2Uub3JnIiwienV1bC50ZW5hbnRzIjp7ImxvY2FsIjoiKiJ9LCJleHAiOjE1Mzc0MTcxOTguMzc3NTQ0fQ.DLbKx1J84wV4Vm7sv3zw9Bw9-WuIka7WkPQxGDAHz7sdfasdf
[scs-stack]
url=https://zuul.otc-scs.t-systems.net/
auth_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwOi8vbWFuYWdlc2Yuc2ZyZG90ZXN0aW5zdGFuY2Uub3JnIiwienV1bC50ZW5hbnRzIjp7ImxvY2FsIjoiKiJ9LCJleHAiOjE1Mzc0MTcxOTguMzc3NTQ0fQ234234wsadfa84wV4Vm7sv3zw9Bw9-WuIka7WkPQxGDAHz7sI
```

## Autohold
This command uses the `.zuul.conf` with auth string 

### Autohold-list
```
zuul-client  --use-config scs autohold-list --tenant scs
+------------+--------+-----------------------------------------+--------------+------------+-----------+---------+
|     ID     | Tenant |                 Project                 |     Job      | Ref Filter | Max Count |  Reason |
+------------+--------+-----------------------------------------+--------------+------------+-----------+---------+
| 0000000000 |  scs   | github.com/opentelekomcloud-scs/testbed | ansible-lint |     .*     |     1     | testing |
| 0000000001 |  scs   | github.com/opentelekomcloud-scs/testbed |    flake8    |     .*     |     1     | testing |
+------------+--------+-----------------------------------------+--------------+------------+-----------+---------+
```

### Autohold
```
zuul-client --use-config scs-stack autohold --tenant scs-stack --project  github.com/opentelekomcloud-scs/zuul-test --job check-repo --reason "debug" --node-hold-expiration 18000
```



### Autohold-delete
```
zuul-client --use-config scs-stack autohold-delete --tenant scs-stack 0000000004
```

## Login to worker nodepool
Since the node of the nodepool is created in a different OTC project, the only possibility to login is in assigning an IP to the instance. This can be done by the configured natgw.

```
-o StrictHostKeyChecking=no
```
The key can be found within the project k8s_development in secrets.


