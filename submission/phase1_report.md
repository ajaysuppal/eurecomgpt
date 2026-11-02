# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->253.7<!--/answer--> | <!--answer:vm_warm_median_ms-->212.7<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2079.6<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->591.2<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1279.3<!--/answer--> | <!--answer:function_warm_median_ms-->653.9<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
vm        cold=  253.7ms  warm_median=  212.7ms
            #############
cloudrun  cold= 2079.6ms  warm_median=  591.2ms
            ####################################
function  cold= 1279.3ms  warm_median=  653.9ms
            ########################################
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. One e2-micro machine handles every request, so if 100 arrive at once it just gets slower. The limit is the size of the machine I chose.
Cloud Run: it automatically starts more copies (instances) of my container when requests pile up, and goes back to zero when idle. The limit is the --max-instances=3 flag I set.
Cloud Function: works the same way, because it runs on Cloud Run underneath. Same limit, the --max-instances=3 flag I set.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM: the most work. I created a custom VPC and subnet, two firewall rules (ports 8080 and 22) with a network tag, the e2-micro instance and a service account role so it could pull the image, then SSHed in, installed Docker and ran the container. I also have to maintain the OS, patches and restarts, and pay for it while it is on. 
Cloud Run: I wrote a Dockerfile, built and pushed the image to Artifact Registry, then one gcloud run deploy, with no servers or networking to manage. 
Cloud Function: least effort. I deployed only the source code with an entry point. Google built the container for me, though I had to grant a Cloud Build role first.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-149752429469.us-central1.run.app
LAST DEPLOYED BY: service-149752429469@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-02T13:47:14.728215Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-149752429469.us-central1.run.app
LAST DEPLOYED BY: Ajay-kumar.suppal@eurecom.fr
LAST DEPLOYED AT: 2026-11-02T13:36:11.430569Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The list shows my function "echo" as an ordinary Cloud Run service, but deployed by gcf-admin-robot, which is Google's Cloud Functions robot, not by me. So a gen-2 function is a Cloud Run service that Google builds for me from my source code. That is why its cold start resembles Cloud Run's, and both start a container from zero. What differs is the deployment path. For Cloud Run I built and pushed the container image myself. For the function I gave only source code and an entry point, and Google built the container. Functions also has its own defaults (256 MB memory, one request at a time per instance).
<!--/answer-->
