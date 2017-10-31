---
layout: default
---

<h1 class="no_toc">DRAFT</h1>

This documentation is still being written. Many sections are empty.

<h1 class="no_toc">Table of Contents</h1>

* TOC
{:toc}


# What is AWS Batch?

From the [website](https://aws.amazon.com/batch/):

> AWS Batch enables developers, scientists, and engineers to easily and efficiently run hundreds of thousands of batch computing jobs on AWS. AWS Batch dynamically provisions the optimal quantity and type of compute resources (e.g., CPU or memory optimized instances) based on the volume and specific resource requirements of the batch jobs submitted. With AWS Batch, there is no need to install and manage batch computing software or server clusters that you use to run your jobs, allowing you to focus on analyzing results and solving problems. AWS Batch plans, schedules, and executes your batch computing workloads across the full range of AWS compute services and features, such as Amazon EC2 and Spot Instances.

AWS Batch uses AWS [EC2 Container Service](https://aws.amazon.com/ecs/) (ECS), meaning your job must be configured as a [Docker](https://www.docker.com/)
container.

## How do I use AWS Batch?

SciComp provides access to AWS Batch in two ways:

* Via the [AWS Command Line Interface (CLI)](https://aws.amazon.com/cli/)
* Via programmatic interfaces such as Python's [boto3](https://boto3.readthedocs.io/en/latest/reference/core/boto3.html). The
  earlier version of this library (`boto`) is deprecated and should not be used.

Console (web/GUI) access to AWS batch is not available to end users at the Center.


# Request access

## Get AWS Credentials

You will need AWS credentials in order to
use AWS Batch. You can get the credentials
[here](https://teams.fhcrc.org/sites/citwiki/SciComp/Pages/Getting%20AWS%20Credentials.aspx).

Initially, these credentials only allow you
to access your PI's S3 bucket. To use the
credentials with AWS Batch, you must request
access to batch.

Request access by emailing **scicomp@fredhutch.org** with the subject
line **Request Access to AWS Batch**.

In your email, **include** the name of your PI.

SciComp will contact you when your access has been granted.

Note that you will not be able to create
compute environments or job queues. If you need a custom compute environment, please contact SciComp.





<div style="display: none;">
scicomp people:

The way to onboard a new user is to do the following:

```
. /app/local/aws-batch-wrapper/env/bin/activate # start virtual env
/app/local/aws-batch-wrapper/onboarding.py # show help message
# For a user with HutchNet ID jblow and PI name peters-u, run:
/app/local/aws-batch-wrapper/onboarding.py jblow peters-u
```

This requires some special permissions, you probably need to
authenticate your command-line session with MFA.

</div>


# Create a Docker image


## Do you need to create your own image?

It depends on what you want to do. If you are using software that is
readily available, there is probably already a Docker image containing
that software. Look around on [Docker Hub](https://hub.docker.com/) to see
if there's already a Docker image available.

The SciComp group is also developing Docker images that contain much
of the software you are used to finding in `/app` on the `rhino`
machines and `gizmo`/`beagle` clusters.

If you've found an existing Docker image that meets your needs, you don't
need to read the rest of this section.


## Getting Started

It's recommended (but not required) that you install
[Docker](https://www.docker.com/) on your workstation (laptop or desktop)
and develop your image on your own machine until it's ready to be deployed.

## Docker Installation Instructions

* [Windows](https://www.docker.com/docker-windows)
* [Mac](https://www.docker.com/docker-mac)
* [Ubuntu Linux](https://www.docker.com/docker-ubuntu)

## Deploy Docker Image

### Create GitHub Account

### Create a Docker Hub Account

### Push your Dockerfile to a GitHub repository

### Create an Automated Build in Docker Hub

# Create a Job Definition

[Job Definitions](https://docs.aws.amazon.com/batch/latest/userguide/job_definitions.html) specify how jobs are to be run. Some of the attributes specified in a job definition include:

* Which Docker image to use with the container in your job
* How many vCPUs and how much memory to use with the container
* The command the container should run when it is started †
* What (if any) environment variables should be passed to the container when it starts †
* Any data volumes that should be used with the container (the compute
  environments provided by SciComp include 1TB of scratch space available
  at `/scratch`).
* What (if any) IAM role your job should use for AWS permissions. This
  is important if your job requires permission to access your PIs
  [S3](https://aws.amazon.com/s3/) bucket.

† = these items can be overridden in individual job submissions.

A custom script is available to generate a job definition template with
Center-specific sections already filled in. The script is called
`get_jobdef_skeleton` and is available on the `rhino` machines.

This script requires that you supply the name of your PI in the format:
lastname + dash + first initial. So if your PI is Jane Doe, you would run
the script as follows:

```
get_jobdef_skeleton -p doe-j > jobdef.json
```

The file `jobdef.json` now contains a partial job definition with the following
information already filled in:

* Mount points and volumes necessary to make available 1TB of scratch
  space in the directory `/scratch`. You are responsible for making sure your
  scripts process all large files in this directory.
* A Job Role which grants your jobs permissions to access your PI's S3 bucket.

Before you can use this file, **you will need to edit it** to fill in
or replace the following values:

* `command`(optional) The command to run when a job starts. This
  can be overridden when submitting a job.
* `environment` (optional) A set of environment variables to pass
  to the containers running your job. Can be overridden when submitting
  a job.
* `memory` The amount of memory (in megabytes) your jobs will need. Defaults
  to 2000 (2GB).
* `vcpus` The number of CPU cores your job will need. Defaults to 1. You
  should only change this if you know that your job code is explicitly
  parallelized (that is, it uses more than one core).
* `jobDefinitionName`  The name of the job definition.
* `image` Change this if you don't want to use the default Docker image
  provided by SciComp.



# Using secrets in jobs

# Using scratch space

If you use a job definition as generated by the
[get_jobdef_skeleton](#create-a-job-definition) script, the container
running your job will have 1TB of writable scratch space available
in the `/scratch` directory. You are responsible for ensuring that all
large files generated in your job will live in this directory; if you
write them elsewhere, you'll run out of space very quickly.

Scratch space is ephemeral; any results you want to keep should be copied
to S3.

Note that AWS Batch may run several containers on the same instance,
in which case they will all use the same scratch space. This means
all jobs on the container must share the 1TB of scratch space. You must also take care to use unique filenames, otherwise files created by one
job could be overwritten by files from other jobs running on the same
instance.

# Using S3 in jobs

The Center's policies require that all of our `S3` buckets be encrypted.
When copying files to or from `S3`, be sure to use the
`--sse AES256` flag in the AWS CLI, or pass the parameter
`ServerSideEncryption='AES256'` if using `boto3`.

If you don't set this flag, you will get cryptic, counterintuitive
error messages (such as `Access Denied`).

# Submit your job

There are currently two ways to submit jobs:

1. via the AWS Command Line Interface (CLI): `aws batch submit-job`.
   Recommended for launching one or two jobs.
1. Using Python's `boto3` library. Recommended for launching
   larger numbers of jobs.

We are looking into various tools to to manage the launching
of large numbers of jobs, and to orchestrate workflows and pipelines.

## Submitting your job via the AWS CLI

The easiest way to submit a job is to generate a JSON skeleton
to pass to [`aws batch submit-job`](https://docs.aws.amazon.com/cli/latest/reference/batch/submit-job.html).
Generate it with this command:

```
aws batch submit-job --generate-cli-skeleton > job.json
```

Now edit `job.json`, being sure to fill in the following fields:

* `jobName` - a unique name for your job, which should include
  your HutchNet ID. . The first character must be alphanumeric, and up to 128 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed.
* `jobQueue` - the name of the job queue to submit to (which
   has the same name as the compute environment that will be used).
   In most cases, you can use the `medium` queue.
* `jobDefinition` The name and version of the job definition to use.
   This will be a string followed by a colon and version number, for
   example: `myJobDef:7`. You can see all job definitions with
   [`aws batch describe-job-definitions`](https://docs.aws.amazon.com/cli/latest/reference/batch/describe-job-definitions.html),
  optionally passing a `--job-definitions` parameter with the name
  of one (or more) job definitions. This will show you each version
  of the specified definition(s).
* If you are using [fetch-and-run](#using-fetch-and-run), do NOT edit
  the `command` field. If you are not using `fetch-and-run` you may
  want to edit this field to override the default command.
* Set the `environment` field to pass environment variables to your jobs.
  This is particularly important when using `fetch-and-run` jobs; these
  require that several environment variables be set. Environment variables
  take the form of a list of key-value pairs with the values `name` and
  `value`, see the following example.

```json
"environment": [
  {
    "name": "FAVORITE_COLOR",
    "value": "blue"
  },
  {
    "name": "FAVORITE_MONTH",
    "value": "December"
  }
]
```  

Now, **delete** the following sections of the file, as we want to
use the default values for them:

* `dependsOn` - this job does not depend on any other jobs.
* `parameters` - we will not be passing parameters to this job.
* `vcpus` in the `containerOverrides` section.
* `memory` in the `containerOverrides` section.
* `retryStrategy` section.

With all these changes made, your `job.json` file will look something
like this:

```json
{
    "jobName": "jdoe-test-job",
    "jobQueue": "medium",
    "jobDefinition": "myJobDef:7",
    "containerOverrides": {
        "command": [
            "echo",
            "hello",
            "world"
        ],
        "environment": [
            {
                "name": "FAVORITE_COLOR",
                "value": "blue"
            },
            {
                "name": "FAVORITE_MONTH",
                "value": "December"
            }
        ]
    }
}
```


Once your `job.json` file has been properly edited, you can
submit your job as follows:

```
aws batch submit-job --cli-input-json file://job.json
```

This will return some JSON that includes the job ID. Be sure and save
that as you will need it to track the progress of your job.

## Submitting your job via `boto3`

### Notes on using Python

* We strongly encourage the use of Python 3. It has been the current
  version of the language since 2008. Python 2 will eventually no longer
  be supported.
* We recommend using [Virtual Environments](http://docs.python-guide.org/en/latest/dev/virtualenvs/),
  particularly [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/),
  to keep the dependencies of your various projects isolated from each
  other.

Assuming `virtualenvwrapper`  and `python3` are installed and
a virtual environment is in use, you'd then
prepare to submit a job via `boto3` as follows:

```
mkvirtualenv --python $(which python3) boto-env
pip install boto3
```

### Submitting your job

Paste the following code into a file called `submit_job.py`:

```python
#!/usr/bin/env python3
"Submit a job to AWS Batch."

import boto3

batch = boto3.client('batch')

response = batch.submit_job(jobName='jdoe-test-job', # use your HutchNet ID
                            jobQueue='medium', # sufficient for most jobs
                            jobDefinition='myJobDef:7', # use a real job definition
                            containerOverrides={
                                "command": ['echo', 'hello', 'world'], # optionally override command
                                "environment": [ # optionally set environment variables
                                    {"name": "FAVORITE_COLOR", "value": "blue"},
                                    {"name": "FAVORITE_MONTH", "value": "December"}
                                ]
                            })

print("Job ID is {}.".format(response['jobId']))

```

Run it with

```
python3 submit_job.py
```

If you had dozens of jobs to submit, you could do it with a `for`
loop in python.


# Monitor job progress

Once your job has been submitted and you have a job ID, you can use it to
retrieve the job status.

The following command will give comprehensive information about your job,
given a job ID:

```
aws batch describe-jobs --jobs 2c0c87f2-ee7e-4845-9fcb-d747d5559370
```
If you are just interested in the status of the job, you can pipe
that command through `jq` (which you may have to
[install](https://stedolan.github.io/jq/download/) first) as follows:


```
aws batch describe-jobs --jobs  2c0c87f2-ee7e-4845-9fcb-d747d5559370 \
| jq -r '.jobs[0].status'
```

This will give you the status (one of `SUBMITTED, PENDING, RUNNABLE,
  STARTING, RUNNING, FAILED, SUCCEEDED`).

# View Job Logs


### On Rhino or Gizmo

On the `rhino` machines or the `gizmo` cluster, there's a quick command
to get the job output. Be sure and use your actual job ID instead of
the example one below:

```
get_batch_job_log 2c0c87f2-ee7e-4845-9fcb-d747d5559370
```

You can also pass a log stream ID (see below) instead of a job ID.


## On other systems

If you are on another system without the `get_batch_job_log` script
(such as your laptop), you can still monitor job logs, but you need to
get the log stream ID first.

To get the log stream for a job, run this command:


```
aws batch describe-jobs --jobs 2c0c87f2-ee7e-4845-9fcb-d747d5559370
```

(Note that you can add additional job IDs (separated by a space) to
get the status of multiple jobs.)

Once a job has reached
the `RUNNING` state, there will be a `logStreamName` field that
you can use to view the job's output. To extract only the `logStreamName`,
pipe the command through [jq](https://stedolan.github.io/jq/download/):

```
aws batch describe-jobs --jobs 2c0c87f2-ee7e-4845-9fcb-d747d5559370 \
jq -r '.jobs[0].container.logStreamName'
```

Once you have the log stream name, you can view the logs:

```
aws logs get-log-events --log-group-name /aws/batch/job \
--log-stream-name jobdef-name/default/522d32fc-5280-406c-ac38-f6413e716c86
```

This outputs other information (in JSON format) along with your log
messages and can be difficult to read. To read it like an ordinary
log file, pipe the command through `jq`:

```
aws logs get-log-events --log-group-name /aws/batch/job \
 --log-stream-name jobdef-name/default/522d32fc-5280-406c-ac38-f6413e716c86 \
| jq -r '.events[]| .message'
```

**NOTE**: `aws logs get-log-events` will only retrieve 1MB worth of
log entries at a time (up to 10,000 entries). If your job has created
more than 1MB of output, read the
[documentation](https://docs.aws.amazon.com/cli/latest/reference/logs/get-log-events.html)
of the `aws batch get-log-events` command to learn about retrieving multiple
batches of log output. (The `get_batch_job_log` script on rhino/gizmo automatically
handles multiple batches of job output, using the
[equivalent command](https://boto3.readthedocs.io/en/latest/reference/services/logs.html#CloudWatchLogs.Client.get_log_events)
in `boto3`.



# Examples

## Using fetch-and-run

## Using a script baked-in to a Docker image

# References

# Future plans

# Questions or Comments

Please direct all questions and feedback to **scicomp@fredhutch.org**.
