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


# What is AWS Batch?

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
[console](https://batch-dashboard.fhcrc.org/) available which displays information
about compute environments, queues, job definitions, and jobs.
Please report any [issues](https://github.com/FredHutch/batch-dashboard/issues/new)
you discover with this console.


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

* `command`(optional) The command to run when a job starts. †
* `environment` (optional) A set of environment variables to pass
  to the containers running your job. †
* `memory` The amount of memory (in megabytes) your jobs will need. Defaults
  to 2000 (2GB). †
* `vcpus` The number of CPU cores your job will need. Defaults to 1. You
  should only change this if you know that your job code is explicitly
  parallelized (that is, it uses more than one core). †
* `jobDefinitionName`  The name of the job definition.
* `image` Change this if you don't want to use the default Docker image
  provided by SciComp.

† = Can be overridden when submitting jobs.


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

**Note**: The method of providing scratch space is going to change.
Check back for updated information.

# Using S3 in jobs

The Center's policies require that all of our `S3` buckets be encrypted.
When copying files to or from `S3`, be sure to use the
`--sse AES256` flag in the AWS CLI, or pass the parameter
`ServerSideEncryption='AES256'` if using `boto3`.

If you don't set this flag, you will get cryptic, counterintuitive
error messages (such as `Access Denied`).

# Submit your job

There are currently three ways to submit jobs:

1. via the AWS Command Line Interface (CLI): `aws batch submit-job`.
   Recommended for launching one or two jobs.
1. Using Python's `boto3` library. Recommended for launching
   larger numbers of jobs.
1. The [run-batch-jobs](#submitting-with-the-run_batch_jobs-utility) utility.

We are looking into various tools to orchestrate workflows and pipelines.

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
  of the specified definition(s). You can also view job
  definitions in the [console](https://batch-dashboard.fhcrc.org).
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
loop in python (or see the next section).

## Submitting with the `run_batch_jobs` utility

A utility available on the `rhino` machines, `run_batch_job`, is designed
to help you launch multiple related jobs.

An example use case might be running a job for every file
in a certain location in your PI's S3 bucket.

Entering the command with the `-h` flag gives you a detailed help message:

```
$ run_batch_jobs -h
usage: run_batch_jobs [-h] [-q QUEUE] [-d JOBDEF] [-n NUMJOBS] [--name NAME]
                      [-f FUNC] [-j JSON] [-x CPUS] [-m MEMORY]
                      [-p PARAMETERS] [-a ATTEMPTS] [-c COMMAND]
                      [-e ENVIRONMENT]

Start a set of AWS Batch jobs. See full documentation at
http://bit.ly/HutchBatchDocs/#submitting-with-the-run_batch_jobs-utility

optional arguments:
  -h, --help            show this help message and exit
  -q QUEUE, --queue QUEUE
                        Queue Name (default: small)
  -d JOBDEF, --jobdef JOBDEF
                        Job definition name:version. (default: hello:2)
  -n NUMJOBS, --numjobs NUMJOBS
                        Number of jobs to run. (default: 1)
  --name NAME           Name of job group (your username and job number will
                        be injected). (default: sample_job)
  -f FUNC, --func FUNC  Python path to a function to customize job, see link
                        above for full docs. Example: myscript.myfunc
                        (default: None)
  -j JSON, --json JSON  Output JSON instead of running jobs. JSON can be used
                        with `aws batch submit-job --cli-input-json`. Makes
                        most sense with `--numjobs 1`. (default: None)
  -x CPUS, --cpus CPUS  Number of CPUs, if overriding value in job defnition.
                        (default: None)
  -m MEMORY, --memory MEMORY
                        GB of memory, if overriding value in job definition.
                        (default: None)
  -p PARAMETERS, --parameters PARAMETERS
                        Parameters to replace placeholders in job definition.
                        Format as a single-quoted JSON list of
                        objects/dictionaries. (default: None)
  -a ATTEMPTS, --attempts ATTEMPTS
                        Number of retry attempts, if overriding job
                        definition. (default: None)
  -c COMMAND, --command COMMAND
                        Command, if overriding job definition. Example:
                       '["echo", "hello world"]' (default: None)
  -e ENVIRONMENT, --environment ENVIRONMENT
                        Environment to replace placeholders in job definition.
                        Format as a single-quoted JSON object/dictionary.
                        (default: None)
```

The real utility of this tool is the `--func` (or `-f`) flag, which
allows you to pass a custom Python 3 function (which you write) that modifies
the job submission information for each job submitted by `run_batch_jobs`.

Write a function that takes two parameters
(call them `obj` and `job_num`).
The first is an object
 containing
job information, and the second is an iteration number
representing the
individual job that is being started. For example, if you called
`run_batch_jobs` with `--numjobs 10`, your function would be called 10
times. The first time through, the value of `job_num` would be 1,
and the last time through, the value of `job_num` would be 10.

Here's an example, which tells the job to print out the job number
(where you can view it in the job logs).

If you save the following code in a file called `testscript.py`, you can
run it with this command line:

```
run_batch_jobs --func testscript.testfunc  --numjobs 10 # start 10 jobs
```

**Contents of testscript.py**:

```python
#!/usr/bin/env python3

"""
An example of a custom function to use with the `run_batch_jobs` utility.
See its full documentation at:
https://fredhutch.github.io/aws-batch-at-hutch-docs/#submitting-with-the-run_batch_jobs-utility
"""

import json


def testfunc(obj, job_num):
    """A function that customizes job input.
    Args:
        obj (dict):    An object containing all information about the job, which may
                       be partially filled in by the command-line options passed
                       to `run_batch_jobs`.
        job_num (int): A number representing this job's index. For example, if you
                       told `run_batch_jobs` to start 10 jobs, this number
                       will be 1 the first time this function is called, and 10 the last time.
    Returns:
        dict: `obj` modified by your custom logic.
    """

    # In this example, we will tell our job to print out a custom message
    # that includes the job iteration number.

    # Let's print out the input object we receive, just to see what it looks like:
    print("Welcome to iteration number {}.".format(job_num))
    print("Before:")
    print(obj)
    print()

    command_object = ["echo", "hello from iteration {}".format(job_num)]
    obj['containerOverrides']['command'] = command_object

    # Let's print out the job object again to see what it looks like after our
    # change:
    print("After:")
    print(obj)
    print()

    # Here we could make any other changes we want to the job object,
    # based on the job iteration number (job_num). For example,
    # if 10 jobs are started, you could do some processing on
    # one of 10 files in S3.

    # now that we have transformed `obj` to our heart's content, we
    # return the modified version:

    return obj

```

The `run_batch_jobs` utility prints out JSON
containing the job name and job ID of each job
it started.

Please report any [issues](https://github.com/FredHutch/aws-batch-wrapper/issues/new)
you discover with the `run_batch_jobs` script.



# Monitor job progress

Once your job has been submitted and you have a job ID, you can use it to
retrieve the job status.

## In the web console

Go to the [jobs table](https://batch-dashboard.fhcrc.org/#jobs_header) in
the console. Paste your job ID or job name into the **Search** box.
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

### In the web console

Go to the [job table](https://batch-dashboard.fhcrc.org/#jobs_header) in the
web console. Paste your job's ID into the **Search** box. Click on
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
