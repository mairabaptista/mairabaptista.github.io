---
title: "Local PySpark + AWS Glue development with Containers"
date: 2023-07-28T12:08:15 -03:00
draft: false
---

## Introduction
Recently I've been tapping into PySpark at work. Plus, since we didn't want to deal with AWS EMR, Glue seemed like a good alternative for deployment, which it is (for our use case)! However, as usual, AWS does not make it simple for the user to develop and test applications. Currently there are three options for developments & testing on a Glue stack:
1. Through the AWS console, by coding your way with a PySpark script. This approach works if you are one of those 10x engineers that are incapable of making mistakes. If you are among mere mortals, like myself, you'll find that not having an IDE and going back and forth from Glue to Cloudwatch are very time consuming ordeals.
2. Use Glue development endpoints. That's AWS solution to the above problem, and it's great! ~~If you're willing to pay for it, that is~~.
3. Local deveploment, which is the neat little thing I'll be writing about today. 

Now, you may be asking yourself, "I can use my favorite IDE, with no extra costs? What's the catch?" There would be a catch if we were dealing with local development the way the ancient mayans used to do: by installing Java, Spark and so on (THE HORROR!). Containers are magic because someone else decided to package all those pesky installation steps for us beforehond! Today, the "someone else" are the folks at AWS themselves, who crafted the Glue images we'll be using today. 

Without further ado, in this article we'll go over the steps required to make a project hosted on AWS Glue using PySpark!

### Prerequisites

This requires Docker to be installed and running. 

## A Makefile
I love makefiles, specially when dealing with long `docker run` commands and I'm being too lazy to create a proper compose file. Plus, it's useful for cutting corners in other ways, such as chaining commands. For now, just create an empty file named `Makefile`.

## AWS Credentials
Since this is not a standalone PySpark tutorial, and we're using AWS Glue, AWS credentials are required. This [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) provided by AWS will guide you through the process.

Let's use the Makefile to store some variables that will be useful. The foisrt one will be the AWS profile nmae you just configured.

```
profile_name := your_profile_name_here
```
## Pulling the image from DockerHub

The image well be using is available on Amazon's [DockerHub](https://hub.docker.com/r/amazon/aws-glue-libs). You can choose the one that'most appropriate for your requirements. For the sake of this tutorial, we'll be using the 4.0 version.

In order to keep our Makefile organized, we are using a variable to store the image tag.
```
image_tag := amazon/aws-glue-libs:glue_libs_4.0.0_image_01
pull-container:
	docker pull $(image_tag)
```
Now, all you need to do to pull the image is to use the folloing command on your terminal:
```
make pull-container
```

## Running the container
There are many ways to use the container we just pulled. We can use it yo launhc a Jupyter Notebook, use VSCode as the development environment, testing and more. For now, only two options are available in this article: `spark-submit` and spark's REPL. `spark-submit` is a command to [launch](https://spark.apache.org/docs/latest/submitting-applications.html) your application and spark's REPL opens up a shell where you can quickly script pysaprk code.

In the ~~far~~ near future I'll update this tutorial with additional wyas to use the Glue container.

### `spark-submit`

Before actually running our container let's create an ETL job file. Since this tutorial is supposed to be about *running* a Glue Job on a local container and not about the ETL job itself, I've included a really simple code sample, named `sample.py`, that creates a spark context, reads a json file from a s3 bucket e prints the resulting DynamicFrame's schema. This should be enough to get you going. 
```python
import sys
from pyspark.context import SparkContext
from awsglue.context import GlueContext

class GluePythonSample:
    def __init__(self):
        self.context = GlueContext(SparkContext.getOrCreate())
        
    def run(self):
        dyf = read_json(self.context, "s3://path_to_your_s3_bucket/file.json")
        dyf.printSchema()

def read_json(glue_context, path):
    dynamicframe = glue_context.create_dynamic_frame.from_options(
        connection_type='s3',
        connection_options={
            'paths': [path],
            'recurse': True
        },
        format='json'
    )
    return dynamicframe

if __name__ == '__main__':
    GluePythonSampleTest().run()
```

Now let's create a Makefile proxy for the `docker run` command:
```
run-container:
	docker run -it \
	--volume ~/.aws:/home/glue_user/.aws
	--volume ${PWD}:/home/glue_user/workspace/ \
	--env AWS_PROFILE=$profile_name \
	--env DISABLE_SSL=true
	--rm \
	--publish 4040:4040 --publish 18080:18080 \
	--name spark-submit-sample \
	$(glue_image_tag) \
	spark-submit \
	--py-files /home/glue_user/workspace/$$JOB_NAME
```

This command  mirrors our local worksapce volume into the container, injects the AWS credentials into it and uses `spark-submit` on our entrypoint file. The entrypoint file, represented by $$JOB_NAME, holds a value entered by the user as argument in the command line, which will be the `sample.py` file.

```
make run-container JOB_NAME=sample.py
```

### REPL

The REPL mode is even simpler, since we're opening a PySpark shell similar to a python shell and not entrypoint is necessary. 
```
run-repl:
	docker run -it \
	--env AWS_PROFILE=$profile_name \
	--env DISABLE_SSL=true
	--rm \
	--publish 4040:4040 --publish 18080:18080 \
	--name -repl \
	$(glue_image_tag) \
	pyspark
```

And for running the container in REPL mode:
```
make run-repl
```

That's it for now!
