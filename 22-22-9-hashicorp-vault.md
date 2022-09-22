---
title: "Use your own Hashicorp Vault instance with Travis CI"
created_at: Thu Sep 22 2022 15:00:00 EDT
author: Michał Rybiński, Montana Mendy
layout: post
permalink: 22-22-9-hashicorp-vault
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

<!-- more --> 

Don’t trust Travis CI with all your secrets needed for the pipeline, at least not too much. If you prefer to manage secrets needed for CI/CD processes in a central Key Management System (KMS), e.g. to be able to rotate them quickly rather than providing secure environment variables to the CI/CD tool, you can now easily connect Travis CI with Hashicorp Vault.

## CI/CD and secrets

In an automated build, test and deployment pipeline often require some secrets - tokens, passwords, certain access credentials - in order to e.g. obtain some artifacts needed for the build and/or push the build artifacts to the target environments. Every CI/CD solution allows build or pipeline creators to manage these secrets with some kind of the solution. In Travis CI it is possible to use environment variables configured via repository settings (stores variable permanently with Travis CI) or to manually [encrypt sensitive data](https://docs.travis-ci.com/user/encryption-keys/) and put an encrypted string into one’s build definition file, `.travis.yml` (uses Travis CI native engine and stored internal keys to decrypt the secret).

Still, in many scenarios a centrally managed Key Management System (KMS) like Hashicorp Vault is used to store, update or invalidate both secrets and configurations. An advantage of using such KMS is a much faster reaction in case of a security incident or establishing a single source of truth for e.g. valid access credentials, configuration values without storing them permanently at CI/CD tool.

Of course, managing one's own secret as a part of ‘limited trust policy’ also serves the purpose of better security!

## Hashicorp Vault

We are pleased to announce that you can now easily use the Hashicorp Vault, one of the popular KMS out there, in connection to Travis CI. Yes, you could have used it already before by providing a certain set of instructions/commands in your build definitions, yet now it’s simpler due to being supported by a convenient `.travis.yml` syntax, which makes it easy to connect to and obtain secrets from the Hashicorp Vault instance.

## How does that work?

Travis CI allows to trigger a build, consisting from one to many concurrent build jobs. Groups of jobs may be sequenced in stages (e.g. build, test, deploy). Each build job is actually an instance of the respective environment (Linux, macOS, Windows…) running for the time required to execute instructions put in `.travis.yml` and after that — disbanded.

An important fact here is that Travis CI build jobs may connect to the public network and when doing so, use a certain range of [IP Addresses](https://docs.travis-ci.com/user/ip-addresses/). This allows KMS administrators to create network rules limiting connections to the KMS system to a specific source, which includes Travis CI jobs.

Once that done and after making sure your Hashicorp Vault instance has the KV engine enabled, Hashicorp Vault instance supports the KV2 API so you can configure connection to the Vault within your build definition and use it to obtain the secrets directly from KMS.

You need to edit your `.travis.yml` adding the connection details, encrypted token allowing access to the Vault and a list of secrets to be obtained. Travis CI will execute connections at the very start of a build job, for which secrets were defined. 

> Please mind that Vault data defined at the root level  of `.travis.yml` will be inherited by all the jobs in the job matrix and available throughout the whole build duration while defining vault data for a single job in the job matrix will keep the connection credentials available only during that job lifetime.

Once connection to the Hashicorp Vault is successful, Travis CI will obtain secrets under paths defined in the `.travis.yml`. If there are no paths to secrets defined in `.travis.yml`, nothing will be acquired. Obtained secrets will be turned into [secure environmental variables](https://docs.travis-ci.com/user/best-practices-security/#steps-travis-ci-takes-to-secure-your-data) with values matching secret value.  From there on you can use these within your build.

## Usage

In order to do so, you will need log into Hashicorp Vault instance and once logged in, obtain [Vault access token](https://www.vaultproject.io/docs/concepts/auth#tokens) (please see also *Considerations* section) Use `travis-cli` to [encrypt the Vault access token](https://docs.travis-ci.com/user/encryption-keys/#usage).

The connection configuration is as easy as:

```yml
os: linux
dist: focal

# vault data at the root level of .travis.yml makes it available for all jobs in the build
vault:
  token: 
    secure: "Your encrypted token goes here"
  api_url: https://your-vault-api.endpoint
  secrets:
    - ns1/project_id/secret_key_a 

script:
#assuming that under /ns1/project_id/secret_key_a a secret with key ‘message’ is present
  - echo $SECRET_KEY_A_MESSAGE ```
`{: data-file=".travis.yml"}
```

Or as a part of one of many jobs:

```yml
os: linux
dist: focal

jobs:
  include:
    - name: “Vault job” # make vault secrets and connection only in this job
      vault:
        token: 
          secure: "Your encrypted token goes here”
        api_url: https://your-vault-api.endpoint
        secrets:
          - ns1/project_id/secret_key_a
      script:
        - echo $SECRET_KEY_A_KEYNAME
    - name: "Another job" # this env variable contains nothing, as this job doesn't connect to Vault
      script: 
        - echo $SECRET_KEY_A_KEYNAME

```

See more examples in our [documentation](https://docs.travis-ci.com/user/hashicorp-vault-integration).

## Considerations

Whenever generating an access token to a Hashicorp Vault instance, carefully consider access rights of the account for which the Vault token is generated. It may be worth it to create a dedicated CI/CD account in Hashicorp Vault with a minimum required scope of access to the certain secrets.

For the initial release, we consciously made it impossible to define Hashicorp Vault access token via repository settings for environment variables. If the token is not provided explicitly in the form of:

```yml
vault:
  token: 
    secure: "Your encrypted token goes here"
```

Travis CI, depending on the exact misstep details: will complain with warnings and/or errors in build config validation section for the job report inability to connect to the Vault and terminate job as failed, recommending verifying credentials simply do not attempt to acquire any secrets from Vault.  We consider access to your Hashicorp Vault instance a sensitive data and decided, at least for now, to enforce strict data securing practice over it. Please consider using access to the Hashicorp Vault only within specific jobs in the job matrix if possible in order to limit potential leakage risks.

Read more information about the integration in our [Hashicorp Vault Integration page](https://docs.travis-ci.com/user/hashicorp-vault-integration).
