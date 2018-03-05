---
layout: default
---

<!--
Short URL for this document, to write on whiteboards, etc:
bit.ly/HutchBatchDocs
-->

<h1 class="no_toc">DRAFT</h1>

This documentation is still being written. Many sections are empty.

<h1 class="no_toc">Table of Contents</h1>

* TOC
{:toc}


# What is AWS Batch??

From the [website](https://aws.amazon.com/batch/):

> AWS Batch enables developers, scientists, and engineers to easily and efficiently run hundreds of thousands of batch computing jobs on AWS. AWS Batch dynamically provisions the optimal quantity and type of compute resources (e.g., CPU or memory optimized instances) based on the volume and specific resource requirements of the batch jobs submitted. With AWS Batch, there is no need to install and manage batch computing software or server clusters that you use to run your jobs, allowing you to focus on analyzing results and solving problems. AWS Batch plans, schedules, and executes your batch computing workloads across the full range of AWS compute services and features, such as Amazon EC2 and Spot Instances.

AWS Batch uses AWS [EC2 Container Service](https://aws.amazon.com/ecs/) (ECS), meaning your job must be configured as a [Docker](https://www.docker.com/)
container.

## How do I use AWS Batch?

SciComp provides access to AWS Batch in two ways:

* Via the [AWS Command Line Interface (CLI)](https://docs.aws.amazon.com/cli/latest/reference/batch/index.html).
* Via programmatic interfaces such as Python's [boto3](https://boto3.readthedocs.io/en/latest/reference/services/batch.html#client). The
  earlier version of this library (`boto`) is deprecated and should not be used.

Access to the AWS Management Console (the web/GUI interface), is not available
to end users at the Center. However, there is a customized, read-only
[dashboard](https://batch-dashboard.fhcrc.org/) available which displays information
about compute environments, queues, job definitions, and jobs.
Please report any [issues](https://github.com/FredHutch/batch-dashboard/issues/new)
you discover with this dashboard.


# Request access

## Get AWS Credentials

You will need AWS credentials in order to
use AWS Batch. You can get the credentials
[here](https://teams.fhcrc.org/sites/citwiki/SciComp/Pages/Getting%20AWS%20Credentials.aspx).

Initially, these credentials only allow you
to access your PI's S3 bucket. To use the
credentials with AWS Batch, you must request
access to Batch.

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
* How many vCPUs and how much memory to use with the container †
* The command the container should run when it is started †
* What (if any) environment variables should be passed to the container when it starts †
* Any data volumes that should be used with the container (the compute
  environments provided by SciComp include 1TB of scratch space available
  at `/scratch`). (**Note**: the process of providing scratch space
  is going to change soon, check back for updated information).
* What (if any) IAM role your job should use for AWS permissions. This
  is important if your job requires permission to access your PI's
  [S3](https://aws.amazon.com/s3/) bucket.

† = these items can be overridden in individual job submissions.



# Using secrets in jobs

# Using scratch space

"Scratch space" refers to extra disk space that your job may
need in order to run. By default, not much disk space is
available (but you have infinite space for input and output
files in [S3](#using-s3-in-jobs)).

The provisioning of scratch space in AWS Batch turns out to
be a very complicated topic. There is no officially supported
way to get scratch space (though Amazon hopes to provide one
in the future), and there are a number of unsupported ways,
each with its own pros and cons.

If you need scratch space, contact SciComp and we can discuss
which approach will best meet your needs.

But first, determine if you **really** need scratch space.
Many simple jobs, where a single command is run on an input file
to produce an output file, can be *streamed*, meaning S3 can
serve as both the standard input and output of the command.
Here's an example that streams a file from S3 to the
command `mycmd`, which in turn streams it back to S3:

```
aws s3 cp s3://mybucket/myinputfile - | mycmd | aws s3 cp --sse AES256 - s3://mybucket/outputfile
```
In the first `aws` command, the `-` means "copy the file
to standard output", and in the second, it means
"copy standard input to S3". `mycmd` knows how to operate
upon its standard input.

By using streams in this way, we don't require any extra disk
space. Not all commands can work with streaming, specifically
those which open files in random-access mode, allowing seeking
to random parts of the file.

If a program does not open files in random-access mode, but
does not explicitly accept input from `STDIN`, or writes more
than one output file, it can still work with streaming
input/output via the use of
[named pipes](https://github.com/FredHutch/s3uploader).

More and more bioinformatics programs can read and write
directly from/to S3 buckets, so this should reduce the need
for scratch space.



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


AWS Batch also supports [array jobs](https://docs.aws.amazon.com/batch/latest/userguide/array_jobs.html), which are collections of related jobs.
Each job in an array job has the exact same command line and
parameters, but has a different value for the
environment variable `AWS_BATCH_JOB_ARRAY_INDEX`.
So you could, for example, have a script which uses
that environment variable as an index into a list of files,
to determine which file to download and process. Array jobs
can be submitted by using either of the methods listed above.

We are looking into additional tools to orchestrate workflows and pipelines.

## Which queue to use?

No matter how you submit your job, you need to choose
a queue to submit to. At the present time, there are two:

* **mixed** - This queue uses a compute environment
  (also called **mixed**) which uses many
  [instance types](https://aws.amazon.com/ec2/instance-types/)
  from the `C` and `M` families. Each of the instance types
  used is one that the Center has high
  [limits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html)
  for in our account.
* **optimal** - This queue uses a compute environment
  (also called **optimal**) which uses the instance type
  `optimal`, meaning Batch will choose from among the
  `C`, `M`, and `R` instance types. While the Center's
  account has high limits for most `C` and `M` types, its
  limits for the `R` types are lower. Batch has no awareness
  of per-account instance limits, so it may try to place
  jobs on `R` instances which could result in longer
  time-to-result.




## Submitting your job via the AWS CLI

The easiest way to submit a job is to generate a JSON skeleton
which can (after editing) be passed to  [`aws batch submit-job`](https://docs.aws.amazon.com/cli/latest/reference/batch/submit-job.html).
Generate it with this command:

```
aws batch submit-job --generate-cli-skeleton > job.json
```

Now edit `job.json`, being sure to fill in the following fields:

* `jobName` - a unique name for your job, which should include
  your HutchNet ID. . The first character must be alphanumeric, and up to 128 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed.
* `jobQueue` - the name of the job queue to submit to (which
   has the same name as the compute environment that will be used).
   In most cases, you can use the `mixed` queue.
* `jobDefinition` The name and version of the job definition to use.
   This will be a string followed by a colon and version number, for
   example: `myJobDef:7`. You can see all job definitions with
   [`aws batch describe-job-definitions`](https://docs.aws.amazon.com/cli/latest/reference/batch/describe-job-definitions.html),
  optionally passing a `--job-definitions` parameter with the name
  of one (or more) job definitions. This will show you each version
  of the specified definition(s). You can also view job
  definitions in the [dashboard](https://batch-dashboard.fhcrc.org).
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
    "jobQueue": "mixed",
    "jobDefinition": "myJobDef:7",
    "containerOverrides": {
        "command": [
            "echo",
            "hello world"
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

Assuming `virtualenvwrapper`  and `python3` are installed, create a virtual environment as follows:

```
mkvirtualenv --python $(which python3) boto-env
pip install boto3 # installs into the virtual environment
```



### Submitting your job

Paste the following code into a file called `submit_job.py`:

```python
#!/usr/bin/env python3
"Submit a job to AWS Batch."

import boto3

batch = boto3.client('batch')

response = batch.submit_job(jobName='jdoe-test-job', # use your HutchNet ID instead of 'jdoe'
                            jobQueue='mixed', # sufficient for most jobs
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
loop in python (but consider using
[array jobs](https://docs.aws.amazon.com/batch/latest/userguide/array_jobs.html)).


# Monitor job progress

Once your job has been submitted and you have a job ID, you can use it to
retrieve the job status.

## In the web dashboard

Go to the [jobs table](https://batch-dashboard.fhcrc.org/#jobs_header) in
the dashboard. Paste your job ID or job name into the **Search** box.
This will show the current status of your job. Click the job ID
to see more details.

## From the command line

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

Note that you can only view job logs once a job has reached the `RUNNING`
state, or has completed (with the `SUCCEEDED` or `FAILED` state).

### In the web dashboard

Go to the [job table](https://batch-dashboard.fhcrc.org/#jobs_header) in the
web dashboard. Paste your job's ID into the **Search** box. Click on
the job ID. Under **Attempts**, click on the **View logs** link.

### On the command line

#### On Rhino or Gizmo

On the `rhino` machines or the `gizmo` cluster, there's a quick command
to get the job output. Be sure and use your actual job ID instead of
the example one below:

```
get_batch_job_log 2c0c87f2-ee7e-4845-9fcb-d747d5559370
```

You can also pass a log stream ID (see below) instead of a job ID.


#### On other systems

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
batches of log output. (The [get_batch_job_log](#on-rhino-or-gizmo) script on rhino/gizmo automatically
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
