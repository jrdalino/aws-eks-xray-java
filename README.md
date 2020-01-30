# myproject-aws-eks-xray-java

## Step 1: Add the SDK as a dependency in your build configuration.
```
*pom.xml*
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-xray-recorder-sdk-core</artifactId>
  <version>1.2.1</version>
</dependency>
```

## Step 2: Add a servlet filter to your deployment descriptor to trace incoming HTTP requests.
```
*web.xml*
<filter>
  <filter-name>AWSXRayServletFilter</filter-name>
  <filter-class>com.amazonaws.xray.javax.servlet.AWSXRayServletFilter</filter-class>
  <init-param>
    <param-name>fixedName</param-name>
    <param-value><<CHANGE_THIS_VALUE>></param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>AWSXRayServletFilter</filter-name>
  <url-pattern>*</url-pattern>
</filter-mapping>
```

## Step 3: Add a policy to the worker node
```
aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

## Step 4: AWS X-Ray DaemonSet - Create a folder and download the daemon
```
~$ mkdir xray-daemon && cd xray-daemon
~/xray-daemon$ curl https://s3.dualstack.ap-southeast-2.amazonaws.com/aws-xray-assets.ap-southeast-2/xray-daemon/aws-xray-daemon-linux-2.x.zip -o ./aws-xray-daemon-linux-2.x.zip
~/xray-daemon$ unzip -o aws-xray-daemon-linux-2.x.zip -d .
```

## Step 5: Create a Dockerfile with the following content.
```
*~/xray-daemon/Dockerfile*
FROM ubuntu:12.04
COPY xray /usr/bin/xray-daemon
CMD xray-daemon -f /var/log/xray-daemon.log &
```

## Step 6: Build the image and Push to new ECR with name: bp-xray-daemon-ecr
```
$ $(aws ecr get-login --no-include-email --region ap-southeast-2)
$ docker build -t bp-xray-daemon-ecr .
$ docker tag bp-xray-daemon-ecr:latest 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/bp-xray-daemon-ecr:latest
$ docker push 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/bp-xray-daemon-ecr:latest
```

## Step 7: Deploy the X-Ray DaemonSet
- Use this as reference: https://eksworkshop.com/intermediate/245_x-ray/daemonset.files/xray-k8s-daemonset.yaml
```
kubectl create -f https://github.com/jrdalino/myproject-aws-eks-xray-java/xray-k8s-daemonset.yaml
```

## Step 8: Validate and View logs
```
$ kubectl describe daemonset xray-daemon
$ kubectl logs -l app=xray-daemon
```

## Step 9: Deploy Application that has X-Ray Configured

## Step 10: Check AWS Console > X-Ray
- View Service Map
- View Traces > Trace Overview
- View Traces > Trace Details
