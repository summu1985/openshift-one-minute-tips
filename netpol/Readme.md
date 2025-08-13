## Openshift Network Policy Starter Kit

This is the repo for the network policy starter kit as described in the LinkedIn post - https://www.linkedin.com/posts/sumit-mukherjee_one-minute-tip-networkpolicies-activity-7354012247712542720-rBXz

This starter kit has following network policies
1. policy-default-deny.yaml -> To deny all incoming and outgoing traffic in all pods
2. policy-allow-backend-egress-to-db.yaml -> To allow outgoing traffic from backend pod to DB pod on port 3306
3. policy-allow-db-ingress-from-backend.yaml -> To allow incoming traffic to DB pod port 3306 from backend pod
4. policy-allow-frontend-egress-to-backend.yaml -> To allow outgoing traffic from frontend to backend pod on port 8080
5. policy-allow-backend-ingress-from-frontend.yaml -> To allow incoming traffic to backend pod on port 8080 from frontend pod

This kit also uses two applications called springboot-frontend and springboot-backend as the frontend and backend pods. 

The application also requires a MySQL DB.

The required container images and MySQL installation and the testing procedure for the network policies are mentioned in the following sections.

## Deploy MySQL

First get the MySQL ephemeral template from openshift and install it in your namespace. 
I have used the namespace summukhe-dev, please alter it as per your need.

```
oc get templates -n openshift | grep mysql-ephemeral
mysql-ephemeral                               MySQL database service, without persistent storage. For more information abou...   8 (3 generated)   3

oc get template mysql-ephemeral -n openshift -o json > mysql-ephemeral.json

sed -i '' 's/"namespace": "openshift",/"namespace": "summukhe-dev",/g' mysql-ephemeral.json

cat mysql-ephemeral.json | grep namespace
        "namespace": "summukhe-dev",
                                "namespace": "${NAMESPACE}"

oc process mysql-ephemeral \
    -p MYSQL_USER=demo \
    -p MYSQL_PASSWORD=<password> \
    -p MYSQL_DATABASE=demodb \
    -p DATABASE_SERVICE_NAME=mysql \
    | oc apply -f -
```

Now verify the DB installation

```
bash-3.2$ oc get pods
NAME             READY   STATUS      RESTARTS   AGE
mysql-1-bz5sm    1/1     Running     0          71s
mysql-1-deploy   0/1     Completed   0          72s

oc exec mysql-1-bz5sm -- mysql -udemo -p
Enter password: ERROR 1045 (28000): Access denied for user 'demo'@'localhost' (using password: NO)
command terminated with exit code 1
```

Please ignore the error. The important aspect is that it responds which means deployment is fine.

Now we will create a secret to store the credentials and some information regarding the DB which will be used in the backend.

I know, I know that secret isn't really secret and we can use configmap etc. But you get the point here. Let's move on.

```
oc create secret generic my-db-creds \
  --from-literal=MYSQL_HOST=mysql \
  --from-literal=MYSQL_PASSWORD=<password> \
  --from-literal=MYSQL_USER=demo \
  --from-literal=MYSQL_DATABASE=demodb \
  --dry-run=client -o yaml | oc apply -f -

oc get secret my-db-creds -o yaml
apiVersion: v1
data:
  MYSQL_HOST: <base 64 encoded host>
  MYSQL_PASSWORD: <base 64 encoded password>
  MYSQL_USER: <base 64 encoded user>
  MYSQL_DATABASE: <base 64 encoded database>
kind: Secret
metadata:
  creationTimestamp: "2025-08-12T10:26:50Z"
  name: my-db-creds
  namespace: summukhe-dev
  resourceVersion: "3358280386"
  uid: 0c450b44-9702-4496-aec9-e7cf3e51c958
type: Opaque
```

## Deploy backend and frontend services

Now let us deploy the backend microservice.

```
oc new-app quay.io/summu85/springboot-backend -l app=backend
oc create route edge --service=springboot-backend
oc set env deploy/springboot-backend --from=secret/my-db-creds
```

## Add a user in backend

```
curl --location --request POST 'https://springboot-backend-summukhe-dev.apps.rm1.0a51.p1.openshiftapps.com/api/user?name=sachin&email=sachin@example.com'
```

## View added user

```
curl --location --request GET 'https://springboot-backend-summukhe-dev.apps.rm1.0a51.p1.openshiftapps.com/api/users'
[{"id":1,"name":"sachin","email":"sachin@example.com"}]bash-3.2$
```

Now let us add the frontend microservice

```
oc new-app quay.io/summu85/springboot-frontend -l app=frontend
oc create route edge --service=springboot-frontend
oc get route springboot-frontend
```

Verify that by default, we are able to fetch the user details via the frontend service.

```
curl https://springboot-frontend-summukhe-dev.apps.rm1.0a51.p1.openshiftapps.com/index | grep -C 5 sachin@example.com

                        </thead>
                        <tbody>
                            <tr>
                                <td>1</td>
                                <td>sachin</td>
                                <td>sachin@example.com</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
```

Now our services are up, time to apply the policies

## Now apply the deny all policy
```
bash-3.2$ cat policy-default-deny.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

oc apply -f policy-default-deny.yaml
```

## Test the backend from frontend pod
```
oc rsh deployment/springboot-frontend
sh-4.4$ curl --max-time 3 https://springboot-backend:8080/api/users/
curl: (28) Resolving timed out after 3000 milliseconds
```
Result - connection failed.

## Test the backend from DB pod
```
bash-3.2$ oc rsh dc/mysql
W0812 18:33:11.683433   31903 warnings.go:70] apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
sh-4.4$ curl --max-time 3 https://springboot-backend:8080/api/users/
curl: (28) Resolving timed out after 3000 milliseconds
sh-4.4$ 
```

## Test that the backend to DB connection from the backend pod itself does not work now
```
bash-3.2$ oc rsh deployment/springboot-backend
sh-4.4$ curl --max-time 3 http://localhost:8080/api/users/
curl: (28) Operation timed out after 3000 milliseconds with 0 bytes received
```
So none of the connections work

## Allow from backend to DB
First check the label on mysql

```
bash-3.2$ oc get pods --show-labels
NAME                                   READY   STATUS    RESTARTS   AGE   LABELS
mysql-1-hpm9t                          1/1     Running   0          44m   deployment=mysql-1,deploymentconfig=mysql,name=mysql
springboot-backend-66bc8c4fd4-jkdn6    1/1     Running   0          30m   app=backend,deployment=springboot-backend,pod-template-hash=66bc8c4fd4
springboot-frontend-655d844fd8-lrcmf   1/1     Running   0          34m   app=frontend,deployment=springboot-frontend,pod-template-hash=655d844fd8
```

Let us first allow egress from backend pod to DB pod port 3306

```
bash-3.2$ cat policy-allow-backend-egress-to-db.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: policy-allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      app: backend
  egress:
    - {}
    - ports:
        - protocol: TCP
          port: 3306
  policyTypes:
    - Egress

bash-3.2$ oc apply -f policy-allow-backend-egress-to-db.yaml
networkpolicy.networking.k8s.io/policy-allow-backend-to-db configured
```

Now apply the network policy to allow incoming connectivity to mysql pod from backend pod
```
bash-3.2$ cat policy-allow-db-ingress-from-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: policy-allow-db-ingress-from-backend
spec:
  podSelector:
    matchLabels:
      name: mysql
  ingress:
    - ports:
        - protocol: TCP
          port: 3306
      from:
        - podSelector:
            matchLabels:
              app: backend
  policyTypes:
    - Ingress

bash-3.2$ oc apply -f policy-allow-db-ingress-from-backend.yaml
networkpolicy.networking.k8s.io/policy-allow-db-ingress-from-backend created
```

## Test that now backend api works 

Now let us check that the communication between backend and DB is working by checking via curl from the backend pod itself
```
bash-3.2$ oc rsh deployment/springboot-backend
sh-4.4$ curl --max-time 3 http://localhost:8080/api/users/
[{"id":1,"name":"sachin","email":"sachin@example.com"}]sh-4.4$
```

## Test that still frontend to backend connection does not work
```
bash-3.2$ oc rsh deployment/springboot-frontend
sh-4.4$ curl --max-time 3 https://springboot-backend:8080/api/users/
curl: (28) Resolving timed out after 3000 milliseconds
sh-4.4$ 
```
## Allow connectivity between frontend and backend

First let us allow egress from frontend to backend pod on port 8080
```
bash-3.2$ cat policy-allow-frontend-egress-to-backend.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: policy-allow-frontend-egress-to-backend
spec:
  podSelector:
    matchLabels:
      app: frontend
  egress:
    - {}
    - ports:
        - protocol: TCP
          port: 8080
  policyTypes:
    - Egress

oc apply -f policy-allow-frontend-egress-to-backend.yaml
networkpolicy.networking.k8s.io/policy-allow-frontend-egress-to-backend created
```
Now let us allow ingress to backend on port 8080 from frontend pod
```
bash-3.2$ cat policy-allow-backend-ingress-from-frontend.yaml 
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: policy-allow-backend-ingress-from-frontend
  namespace: summukhe-dev
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - ports:
        - protocol: TCP
          port: 8080
      from:
        - podSelector:
            matchLabels:
              app: frontend
  policyTypes:
    - Ingress

bash-3.2$ oc apply -f policy-allow-backend-ingress-from-frontend.yaml
networkpolicy.networking.k8s.io/policy-allow-backend-ingress-from-frontend created
```

Now let us test that communication between frontend and backend pods also work.
```
bash-3.2$ oc rsh deployment/springboot-frontend
sh-4.4$ curl --max-time 3 http://springboot-backend:8080/api/users/
[{"id":1,"name":"sachin","email":"sachin@example.com"}]
```
## Final testing
Let us verify that only the allowed communication works.
Let us check if we can call the frontend pod from backend

```
bash-3.2$ oc rsh deployment/springboot-backend
sh-4.4$ curl --max-time 3 http://springboot-frontend:8080/index
curl: (28) Connection timed out after 3001 milliseconds
```
while the call from openshift ingress is allowed (This is by default allowed in Openshift namespaces)

```
curl https://springboot-frontend-summukhe-dev.apps.rm1.0a51.p1.openshiftapps.com/index | grep -C 5 sachin@example.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2225    0  2225    0     0   2189      0 --:--:--  0:00:01 --:--:--  2192
                        </thead>
                        <tbody>
                            <tr>
                                <td>1</td>
                                <td>sachin</td>
                                <td>sachin@example.com</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
```
In this example, you learnt how to lockdown communication in pods in Openshift and only allow required communication.

Secure by design > secure by hope !!
