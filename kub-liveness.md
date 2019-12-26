---
layout: page
---
# Liveness in Kubernetes using Helm

In recent engagement with a client, we implemented concept of liveness health-check to check if application services are up. In case liveness check fails, application pod gets restarted.

Here is brief summary of overall logic one can used to achieve the same. Idea is to execute a script (shell/python etc.) on a regular interval of time to check health of a pod. Script can return 0 or 1. If 1 is returned from this script, liveness is treated to be failed and pod gets restarted.

One can define health-check to be performed depending on the requirement. Health-check can be simply to verify a service e.g. DB service availability before bringing the application container up.

In our case, we wanted to check if:

* Application service are up by performing curl operation inside the container on given URL/Port. If HTTP response code does not match with the expected user defined code, liveness check is treated as failed which triggers restart of the pod.


Let's have a look on what needs to be done to achieve this.


### Step 1: Build your docker image with liveness script in place


Build your application docker image using  the sample liveness  script  provided .The script will be copied to the container image and is present in /tmp directory inside pod. 


Note : Liveness Script makes use of curl - so please make sure image is build with curl package installed inside. Refer to  the following sample docker file as an example


Sample Docker file: Assuming liveness script name is liveness.sh

```
FROM nginx
RUN  apt-get  update
RUN  apt-get  install -y curl 
COPY liveness.sh /tmp/
RUN chmod +x /tmp/liveness.sh
#Following is needed to be adjusted as per the user with which docker runs
RUN chown -R nobody:nobody /tmp/liveness.sh
CMD ls && /bin/bash
EXPOSE 8080
EXPOSE 443
```

**Liveness Script :**

```
#!/bin/bash


#define URL along with Port e.g. localhost:8080, localhost:8080/int
URL_CHECK="localhost:8080"
#define expected HTTP response code expected from above URL e.g. 200
RESP_CODE=200

# following command checks for service using curl command
CURL_CHECK=`timeout 20 curl -sL -w "%{http_code}\\n" "$URL_CHECK" -o /dev/null`


if [ "$CURL_CHECK" -eq "$RESP_CODE" ]
then
        exit 0;
else
        echo "Liveness check failed for mentioned URL: $URL_CHECK";
        exit 1;
fi

```

The  variables  **URL_CHECK** and **RESP_CODE** defined in the liveness script will vary from application to application and should be changed  before building the image (copying script inside the image)

Let’s assume your application listens on port 8080 and returns code 200 for successful curl request then, 


**URL_CHECK** variable needs to be set to “localhost:8080” RESP_CODE variable needs to be set to 200


### Step 2: Modify values.yaml to execute liveness check

Include following section to enable  health-check/liveness check
```
healthCheck:
  liveness_exec:
    - /tmp/liveness.sh
  initialDelaySeconds: 90
  timeoutSeconds: 25
```

### Step 3: Deploy Application using the image build & pushed in step#1 and values.yaml health-check section inclusion (step#2)


References: <br />
https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes
<br />
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
