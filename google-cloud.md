Google Cloud
============

This page contains my notes on getting started with Google Cloud.

Starting an instance
--------------------

Start a VM via the web interface as documented [here](https://cloud.google.com/compute/docs/create-linux-vm-instance). Only try the CLI on once that works successfully.

Everything you do is associated with a project, as I only have one project, called "My First Project", that was created by doing the introductory walk-thru of creating one's first VM, everything I do below is associated with that project.

Go to _Compute Engine_ / _Settings_ and set your default zone to `europe-west4` otherwise it always defaults to somewhere in the US for everything.

On pressing _Create_ for my first instance, startup failed - this was shown by a red dot in the _Status_ column but as there was an in-progress icon rotating away I thought this was just to indicate it wasn't yet started. But hovering over the red dot showed:

> e2 instances do not support onHostMaintenance=TERMINATE unless they are preemptible

Googling for this came up with nothing better than just to do everything again, i.e. no need to change anything. And indeed second time and subsequently things worked without issue.

Web console crashes
--------------------

I found that the web console could get into a state where it just stopped working, the main content area failed to display anything when you clicked on things in the left-hand panel but there were no obvious errors (unless you opened the Javascript console). Just close the tab and open <https://console.cloud.google.com/> in a new tab.

My impression is that the web consoles are bug ridden - fields that should be editable only become editable if you click into other fields and then pack, old tabs remember old settings even if you click between sections and sometimes (as noted above) the whole thing just breaks.

Command line interface
-----------------------

Digested from [here](https://cloud.google.com/sdk/docs/install-sdk#deb). The CLI tool is called `gcloud`.

### Install gcloud

```
$ sudo apt-get install apt-transport-https ca-certificates gnupg
```

Note: these were already all installed.

```
$ echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
$ curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
$ sudo apt-get update
$ sudo apt-get install google-cloud-cli
```

Then initialize things:

```
$ gcloud init
```

First it opens a browser and you have to select you account and grant the `gcloud` relevant permissions, then it's back to the command line to complete the process.

I chose `europe-west4-a` as my zone as the region `europe-west4` is the only European region that contains the GPU instances I'm interested in. I don't know what difference it makes choosing `europe-west4-a` vs the `b` and `c` zones.

Note: the default zone and region you specify here is just for `gcloud` actions, it's independent of the zone you set in the web console under _Compute Engine_ / _Settings_.

### Launch an instance

Digested from [here](https://cloud.google.com/compute/docs/instances/create-start-instance#gcloud). Find an image:

```
$ gcloud compute images list --filter ubuntu-os-cloud
NAME                                  PROJECT          FAMILY                   DEPRECATED  STATUS
ubuntu-1804-bionic-v20220505          ubuntu-os-cloud  ubuntu-1804-lts                      READY
ubuntu-2004-focal-v20220419           ubuntu-os-cloud  ubuntu-2004-lts                      READY
...
```

If you use the family name rather than the image name then you should get the latest version for that family.

List machine types:

```
$ gcloud compute machine-types list --zones=europe-west4-a,europe-west4-b,europe-west4-c
```

Start an instance:

```
$ gcloud compute instances create foobar-cli --image-project=ubuntu-os-cloud --image-family=ubuntu-2004-lts --machine-type=e2-medium
Created [https://www.googleapis.com/compute/v1/projects/verdant-wares-350212/zones/europe-west4-a/instances/foobar-cli].
NAME          ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
foobar-cli    europe-west4-a  e2-medium                  10.164.0.3   34.147.87.232  RUNNING
```

`foobar-cli` is the name I gave the instance - I created another one called `foobar-web` via the web interface so, I could compare and confirm that I'd created something fairly identical.

Ssh to the instance:

```
$ gcloud compute ssh foobar-cli
```

First time around, this creates the files `google_compute_engine`, `google_compute_engine.pub` and `google_compute_known_hosts` under `~/.ssh`.

Also, somewhat oddly, it creates a user with an identical name to the local user, i.e. it takes the value of `$USER` and creates a corresponding user on the remote machine. You can even do:

```
$ gcloud compute ssh foobar@foobar-cli
```

And it will create an additional user called `foobar` and log you in as that user.

To see your keys, go to _Compute Engine_, click _Metadata_ and then select the _SSH KEYS_ tab.

Once a key and username is associated with your project, you can just use normal `ssh`. Go to _Compute Engine_ / _VM instances_, find your VM and copy the _External IP_ value, then:

```
$ GCP_IP=34.147.87.232
$ ssh -i ~/.ssh/google_compute_engine -o UserKnownHostsFile=~/.ssh/google_compute_known_hosts $USER@$GCP_IP
```

You can get all details for an instance with:

```
$ gcloud compute instances describe instance-cloud3
```

Or just get e.g. its public IP address:

```
$ gcloud compute instances describe instance-cloud3 --format='value(networkInterfaces.accessConfigs[0].natIP)'
35.204.67.223
```

I launched an instance via the CLI and via the web console and described both to see if I was missing anything from my CLI arguments - there were no interesting differences between the two.

To delete an instance:

```
$ gcloud compute instances delete foobar-cli
```

This is very slow - it takes around 50s to complete - use `delete --quiet` to delete without prompting for confirmation.

### Google GPUs

For pricing see [GPU priving](https://cloud.google.com/compute/gpus-pricing) - initially, it looked like there were no prices here but it was just that the price table is loaded via a slow AJAX call.

T4: $0.35 p/h  
P4: $0.65 p/h  
P100: $1.60 p/h  
V100: $2.55 p/h  
A100: $2.93 p/h

When I first tried to launch a GPU instance, it failed due to not having a quota for GPUs. Finding the relevant quota setting was bizarrely involved. In the end it's just:

* Go to _IAM_ / _Quota_ and in the filter area paste in `GPUs (all regions)` - don't try selecting _Quota_ as a filter option and expecting this to pop up in the list of quotas to choose from.

It turns out you can't edit quotas unless you click _ACTIVATE_ at the top of the page for a full account rather than a trial account.

As with AWS, you have to justify your request (I don't know what they mean by your "service provider" - I assume this just means GCP).

My request was rejected less than a minute later - this could be seen by going to _IAM_ / _Quotas_ and selecting the _INCREASE REQUESTS_ tab.

Blender on a compute optimized instance
---------------------------------------

The biggest machines to which I have access are 8 vCPU compute optimized machines:

After starting a `c2-standard-8` instance:

```
$ GCP_IP=34.141.222.155
$ scp -i ~/.ssh/google_compute_engine -o UserKnownHostsFile=~/.ssh/google_compute_known_hosts render-packed.blend $USER@$GCP_IP:.
$ ssh -i ~/.ssh/google_compute_engine -o UserKnownHostsFile=~/.ssh/google_compute_known_hosts $USER@$GCP_IP
```

```
$ version=3.1.2
$ url=$(curl -s -D- -L https://www.blender.org/download/release/Blender${version%.*}/blender-$version-linux-x64.tar.xz -o /dev/null | sed -n 's/^refresh:.*url=\(.*\.xz\).*/\1/p')
$ curl -sL $url -o blender.tar.xz
$ mkdir blender
$ time tar -xf blender.tar.xz --strip-components=1 -C blender
```

```
$ sudo apt install libgl-dev libxrender-dev libxi-dev
$ time blender/blender -b render-packed.blend --python-expr 'import bpy ; bpy.data.scenes["Scene"].cycles.samples = 128' -E CYCLES -s 1 -e 5 -o //result/ -f 1
...
real	2m1.780s
user	16m2.659s
sys	0m1.288s
```

So 2m for 128 samples, cf with 27s for 512 samples on an AWS V100 instance.

The C2 instances have Intel CPUs, the C2D ones have AMD cores. Let's try again with a C2D instance:

```
$ gcloud compute instances create instance-c2d-standard-8 --image-project=ubuntu-os-cloud --image-family=ubuntu-2004-lts --machine-type=c2d-standard-8
Created [https://www.googleapis.com/compute/v1/projects/verdant-wares-350212/zones/europe-west4-a/instances/instance-c2d-standard-8].
NAME                     ZONE            MACHINE_TYPE    PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
instance-c2d-standard-8  europe-west4-a  c2d-standard-8               10.164.0.8   34.147.87.232  RUNNING
$ GCP_IP=34.147.87.232
$ scp -i ~/.ssh/google_compute_engine -o UserKnownHostsFile=~/.ssh/google_compute_known_hosts render-packed.blend $USER@$GCP_IP:.
```

This failed as it turns out Google really do reuse a limited set of IP addresses and I'd hit one that I'd already got for another instance, so avoid the dire warnings:

```
$ ssh-keygen -f ~/.ssh/google_compute_known_hosts -R $GCP_IP
```

Then:

```
$ scp -i ~/.ssh/google_compute_engine -o UserKnownHostsFile=~/.ssh/google_compute_known_hosts render-packed.blend $USER@$GCP_IP:.
```

The same Blender setup as before and...

```
$ time blender/blender -b render-packed.blend --python-expr 'import bpy ; bpy.data.scenes["Scene"].cycles.samples = 128' -E CYCLES -s 1 -e 5 -o //result/ -f 1
...
real	2m11.508s
user	17m17.099s
sys	0m0.720s
$ exit
```

```
$ time gcloud compute instances delete instance-c2d-standard-8 
```

So slower than the Intel instance.