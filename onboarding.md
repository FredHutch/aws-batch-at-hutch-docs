# Onboarding to AWS Batch

**NOTE**: This documentation is for SciComp staff only. Other users do
not have the permissions required to onboard new users and following
these instructions will result in errors.

## Overview

The way to onboard a new user is to do the following on one of the `rhino` nodes:


```
. /app/local/aws-batch-wrapper/env/bin/activate # start virtual env
/app/local/aws-batch-wrapper/onboarding.py # show help message
# For a user with HutchNet ID jblow and PI name peters-u, run:
/app/local/aws-batch-wrapper/onboarding.py jblow peters-u
# If the user has a 'div' bucket instead of a 'pi' bucket, do this
# (omit the fh-div-):
/app/local/aws-batch-wrapper/onboarding.py jblow sr-genomics
```

This requires some special permissions, you probably need to
authenticate your command-line session with MFA.

The `onboarding.py` code can be found in the 
[aws-batch-wrapper](https://github.com/FredHutch/aws-batch-wrapper)
repository.

