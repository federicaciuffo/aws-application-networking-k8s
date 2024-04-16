# Share Kubernetes Gateway (VPC Lattice Service Network) between different AWS accounts

[AWS Resource Access Manager](https://aws.amazon.com/ram/) (AWS RAM) helps you share your resources across AWS Accounts, within your Organization or Organizational Units (OUs). 

While VPC Lattice support 2 types of resource sharing (VPC Lattice Services and  Service Networks), today the AWS Gateway API Controller only supports sharing VPC Lattice Service Networks.



**Steps**

Let's build an example where Account B (*sharer account*) shares its Service Network with Account A (*sharee account*), and Account A can access all Kubernetes `services` (VPC Lattice Target Groups) and Kubernetes `httproutes`(VPC Lattice Services) within this sharer account's service network.

1. Create a full connectivity setup example (include a gateway, a service and a httproute ) in the Account B (*sharer account*): 
```bash
kubectl apply -f files/examples/second-account-gw1-full-setup.yaml
```

2. Open the AWS RAM console in Account B (*sharer account*), create a `VPC Lattice Service Networks` type resource sharing, share the service network that created from previous step's gateway (`second-account-gw1`)  (You could check
the VPC Lattice console to get the resource arn, service network name should also be `second-account-gw1`)

3. Open the Account A (*sharee account*)'s AWS RAM console and accept the Account B's Service Network sharing invitation in the *"Shared with me"* section .

4. Load the Account A's AWS credentials in you command line, and switch to accountA's context.
```bash
kubectl config use-context <accountA cluster>
``` 

5. Apply the same `second-account-gw1` Account A (*sharee account*)'s cluster:
```bash
kubectl apply -f files/examples/second-account-gw1-in-primary-account.yaml
```

6. All done, you could verify Service Network (Gateway) sharing by: Attach to any pod in Account A's cluster, do `curl <vpc lattice service dns for 'second-account-gw1-httproute'>`, it should be able to get correct response *"
   second-account-gw1-svc handler pod" *
