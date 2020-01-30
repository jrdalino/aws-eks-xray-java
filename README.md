# myproject-aws-eks-xray-java

## Step 1: Add the SDK as a dependency in your build configuration.
```
*pom.xml*
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-xray-recorder-sdk-core</artifactId>
  <version>1.2.1</version>
</dependency>

OR

<dependency> 
     <groupId>com.amazonaws</groupId> 
     <artifactId>aws-xray-recorder-sdk-spring</artifactId> 
     <version>2.4.0</version> 
</dependency>
```

## Step 2: Adding a Tracing Filter to your Application (Spring) to trace incoming HTTP requests.
- For Spring, add a Filter to your WebConfig class. Pass the segment name to the AWSXRayServletFilter constructor as a string.
- Example src/main/java/myapp/WebConfig.java - Spring
```
package myapp;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import javax.servlet.Filter;
import com.amazonaws.xray.javax.servlet.AWSXRayServletFilter;

@Configuration
public class WebConfig {

  @Bean
  public Filter TracingFilter() {
    return new AWSXRayServletFilter("Scorekeep");
  }
}
```

## Step 3: Annotating Your Code or Implementing an Interface. Example:
- The following code extends the abstract class AbstractXRayInterceptor.
```
@Aspect
@Component
public class XRayInspector extends AbstractXRayInterceptor {    
    @Override    
    protected Map<String, Map<String, Object>> generateMetadata(ProceedingJoinPoint proceedingJoinPoint, Subsegment subsegment) throws Exception {      
        return super.generateMetadata(proceedingJoinPoint, subsegment);    
    }    
  
  @Override    
  @Pointcut("@within(com.amazonaws.xray.spring.aop.XRayEnabled) && bean(*Controller)")    
  public void xrayEnabledClasses() {}
  
}
```
- The following code is a class that will be instrumented by X-Ray.
```
@Service
@XRayEnabled
public class MyServiceImpl implements MyService {    
    private final MyEntityRepository myEntityRepository;    
    
    @Autowired    
    public MyServiceImpl(MyEntityRepository myEntityRepository) {        
        this.myEntityRepository = myEntityRepository;    
    }    
    
    @Transactional(readOnly = true)    
    public List<MyEntity> getMyEntities(){        
        try(Stream<MyEntity> entityStream = this.myEntityRepository.streamAll()){            
            return entityStream.sorted().collect(Collectors.toList());        
        }    
    }
}
```

## Step 4: Add a policy to the worker node
```
aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

## Step 5: AWS X-Ray DaemonSet - Create a folder and download the daemon
```
~$ mkdir xray-daemon && cd xray-daemon
~/xray-daemon$ curl https://s3.dualstack.ap-southeast-2.amazonaws.com/aws-xray-assets.ap-southeast-2/xray-daemon/aws-xray-daemon-linux-2.x.zip -o ./aws-xray-daemon-linux-2.x.zip
~/xray-daemon$ unzip -o aws-xray-daemon-linux-2.x.zip -d .
```

## Step 6: Create a Dockerfile with the following content.
```
*~/xray-daemon/Dockerfile*
FROM ubuntu:12.04
COPY xray /usr/bin/xray-daemon
CMD xray-daemon -f /var/log/xray-daemon.log &
```

## Step 7: Build the image and Push to new ECR with name: bp-xray-daemon-ecr
```
$ $(aws ecr get-login --no-include-email --region ap-southeast-2)
$ docker build -t bp-xray-daemon-ecr .
$ docker tag bp-xray-daemon-ecr:latest 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/bp-xray-daemon-ecr:latest
$ docker push 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/bp-xray-daemon-ecr:latest
```

## Step 8: Deploy the X-Ray DaemonSet
- Use this as reference: https://eksworkshop.com/intermediate/245_x-ray/daemonset.files/xray-k8s-daemonset.yaml
```
kubectl create -f https://github.com/jrdalino/myproject-aws-eks-xray-java/xray-k8s-daemonset.yaml
```

## Step 9: Validate and View logs
```
$ kubectl describe daemonset xray-daemon
$ kubectl logs -l app=xray-daemon
```

## Step 10: Deploy Application that has X-Ray Configured

## Step 11: Check AWS Console > X-Ray
- View Service Map
- View Traces > Trace Overview
- View Traces > Trace Details
