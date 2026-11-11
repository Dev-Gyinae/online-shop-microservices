# online-shop-microservices


pending issues:
 - k8s having issues pulling image 

The issue is with resource allocation the VM is currently running 2gb ram and these services need at least 4gb and 8gb to run.
The nodes are currently not getting enough space allocation.

Solution:
- I built a new vm with 8gig ram and 1gb volume so the deployment is running now
