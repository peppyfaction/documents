---
layout: page
---
# Liveness in Kubernetes using Helm

In recent engagement with a client, we implemented liveness for stateful applications (using OpenEBS PVCs).


Here is brief summary of overall logic one can used to achieve the same. Idea is to execute a script (shell/python etc.) on a regular interval of time to check health of a container. Script can return 0 or 1. If 1 is returned liveness is treated to be failed and container gets restarted.


One can define health-check to be performed depending on the requirement. Health-check can be simply to verify a service e.g. DB service availability before bringing the application container up.


In our case, we wanted to check if:


Application service are up by performing curl operation inside the container on given URL/Port. If HTTP response code does not match with the expected user defined code, liveness check is treated as failed. 


Mounted persistent storage volume is in healthy state by performing a write operation on the mounted PVC. If write operations fails, liveness check is treated as failed.


In case liveness check fails, container gets restarted.


Let's have a look on what needs to be done to achieve this.


Step 1: Build your docker image with liveness script in place


Build your application docker image using  the sample liveness  script  provided .The script will be copied to  the container image and will be present in /tmp directory inside pod. 


Note : Liveness Script makes use of curl - so please make sure image is build with curl package installed inside. Refer to  the following sample docker file as an example


Sample Docker file: Assuming liveness script name is liveness.sh

```
FROM registry.mckinsey.com/gold-images/nginx:1.10.3
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


#define mount path and http host URL to be checked
MOUNT_PATH="/tmp" 
#define URL along with Port e.g. localhost:8080, localhost:8080/int
URL_CHECK="localhost:8080"
#define expected HTTP response code expected from above URL e.g. 200
RESP_CODE=200


TMP_FILE="${MOUNT_PATH}/tmpfile"
timeout 10  echo "7" >  $TMP_FILE
EXT_CODE_FS=1;
EXT_CODE_URL=1;


TEST_OUT=`timeout 10 cat $TMP_FILE`


if [ "$TEST_OUT" -eq  "7" ]
then
        EXT_CODE_FS=0;
else
        echo "Liveness failed for mounted FS";
        EXT_CODE_FS=1;
fi


CURL_CHECK=`timeout 20 curl -sL -w "%{http_code}\\n" "$URL_CHECK" -o /dev/null`


if [ "$CURL_CHECK" -eq "$RESP_CODE" ]
then
        EXT_CODE_URL=0;
else
        echo "Liveness check failed for mentioned URL: $URL_CHECK";
        EXT_CODE_URL=1;
fi


if [ $EXT_CODE_FS -eq 0 -a $EXT_CODE_URL -eq 0 ]
then
        EXT_CODE=0
else
        EXT_CODE=1
fi


timeout 10 rm -f $TMP_FILE
exit $EXT_CODE
```

The  variables MOUNT_PATH, URL_CHECK and RESP_CODE defined in the liveness script will vary from application to application and should be changed  before building the image (copying script inside the image)


MOUNT_PATH variable should be exactly the same as defined in the values.yaml file.


For e.g - let’s assume following configuration has been defined in your values.yaml for persistence storage 

```
config:
  volumeMounts:
  - mountPath: /data/test
    resources:
      storageClass: openebs-nfs
      storage: 500Mi 
      accessMode: "ReadWriteMany" 
```  
    
In this case MOUNT_PATH variable should  be set to \“/data/test\”

Let’s assume your application listens on port 8080 and returns code 200 for successful curl request then, 


URL_CHECK variable needs to be set to “localhost:8080” RESP_CODE variable needs to be set to 200


Step 2: Modify values.yaml to execute liveness check

Include following section to enable  health-check/liveness check
```
healthCheck:
  liveness_exec:
    - /tmp/liveness.sh
  initialDelaySeconds: 90
  timeoutSeconds: 25
```

Step 3: Deploy Application using the image build & pushed in step#1 and values.yaml health-check section inclusion (step#2)


References:j
https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
