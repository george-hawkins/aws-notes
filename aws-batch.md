AWS Batch
---------

This page contains notes made while trying AWS Batch for the first time.

Search for "Batch" (oddly, searching for "AWS Batch" doesn't work) in the AWS web interface search bar.

### Compute environment

Click _Compute environments_, then the _Create_ button.

* Compute environment name: my-first-compute-environment
* Provisioning model: Spot
* Maximum % on-demand price: 50 (AWS has a [spot instance advisor](https://aws.amazon.com/ec2/spot/instance-advisor/) that shows the savings you can typically expect).
* Maximum vCPUs: 4
* Allowed instance types: Optimal

You can choose from various provisioning environments, e.g. Fargate, but Fargate doesn't support GPU usage, see this [issue](https://github.com/aws/containers-roadmap/issues/88).

That was it, then _Create compute environment_. It'll then complain that you didn't specify a _Spot fleet role_ - the _Learn more_ link takes you straight to the relevant information. You don't need to be the root user to create this role. The instructions point you to a page where you...

Select the radio box for _AWS service_ (selected by default) and, rather than the basic _EC2_ radio box, select _EC2_ from the dropdown and then it shows more roles, select _EC2 Spot Fleet Tagging_.

Then _Next_ twice and enter "AmazonEC2SpotFleetTaggingRole" in the _Role name_ field and then _Create role_.

Now, back on the _Create compute environment_ page, click the refresh icon beside the _Spot fleet role_ field and select the newly created role from the dropdown. Now, _Create compute environment_ works.

### Job queue

Now go to _Job queues_ and then _Create_.

* Job queue name: my-first-job-queue
* Leave everything else alone and at the bottom select the just created compute environment from the _Select a compute environment_ dropdown and then tick its radio button.
* Hit _Create_.

### Getting going with Docker

Try out the `amazonlinux:latest` image:

```
$ mkdir docker-experiments
$ cd docker-experiments
$ docker run -it amazonlinux:latest /bin/bash
```

Create an image for your batch jobs:

```
$ curl -L -O https://raw.githubusercontent.com/awslabs/aws-batch-helpers/master/fetch-and-run/fetch_and_run.sh
$ chmod 755 fetch_and_run.sh 
$ cat > Dockerfile << 'EOF'
FROM amazonlinux:latest
RUN yum -y install unzip aws-cli
ADD fetch_and_run.sh /usr/local/bin/fetch_and_run.sh
WORKDIR /tmp
USER nobody
ENTRYPOINT ["/usr/local/bin/fetch_and_run.sh"]
EOF
$ docker build -t foobar .
```

Then try it out:

```
$ docker run foobar
$ docker run -it --entrypoint /bin/bash foobar
bash-4.2$ fgrep image_file /etc/image-id 
image_file="amzn2-container-raw-2.0.20220426.0-x86_64"
```
