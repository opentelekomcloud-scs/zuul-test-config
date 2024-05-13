# ZUUL Config
Zuul Tenant configured in zuul-config Repo with poniter to this Repo as a config repo and zuul-test as the project repo:

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

Jobs within this repo have enhanced rights. They should be reviewed with care.


