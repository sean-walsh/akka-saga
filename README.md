# akka-transactional

## Implementation notes

This is an exploration of how to use akka cluster to perform in a transactional manner. The behavior
should mimic a Saga or what used to be called an Akka Process Manager Pattern. The difference is that
this type of saga is co-located in the same cluster as the business functionality it interacts with,
making it more realtime in nature.

The saga itself is a long-running Akka persistent actor, sharded across the cluster. The saga will 
remain active until either all transactions are commited OR all transactions are rolled back due to 
any error or business exception.

Interestingly, any business persistent actor participating in a saga will essentially be "locked"
during the saga, meaning that the actor may not participate in any other sagas until the initial 
one has completed. I use Akka Stash for this.

Patterns used here are event sourcing of all business components, including the sage as well as 
CQRS (just commands for now).

There is full integration into the Lightbend Enterprise Suite 2.0 for visibility of behaviors.

## The use case

This is a use case I heard not once but twice in the financial sector. It involves a batch of bank
account transactions, in this case withdrawals and deposits. If any single one of the transactions
fail, the entire batch must fail. This ACID type transaction pattern works fully
within an Akka cluster and does not need Kafka, though it would be a nice addition
to add a Kafka option for integration across completely disparate systems. For
high performance applications, using the eventlog instead of Kafka should provide
near real time performance.


## use case 2 --todo

Have completely separate functionality, such as anomalies cancel
a transaction within a given time window. It would be possible to co-locate
a security type application within the same cluster as the transactional
application(s) such as bank account for realtime inspection and interuption
of tranacations. This can also optionally utilize Kafka instead of the event
log or in addition for nice decoupled integration across services. 

## Deployment

This is completely ready to deploy using our enterprise suite tooling onto Kubernetes Minikube. Change into the
project folder and do the following for Mac OS. You must have or obtain Lightbend credentials to run this project.
See project/plugins.sbt. For credentials, please go here: https://www.lightbend.com/contact

1. Install VirtualBox, kubectl and minikube. See: https://kubernetes.io/docs/tasks/tools/install-minikube/
2. minikube start --cpus 3 --memory 4000
3. minikube addons enable ingress
4. kubectl apply -f rbac.yaml
5. eval $(minikube docker-env)
6. sbt docker:publishLocal
7. rp generate-kubernetes-resources "akka-saga:0.1.0" \
     --generate-pod-controllers --generate-services \
     --pod-controller-replicas 4 | kubectl apply -f -
     
8. rp generate-kubernetes-resources \
     --generate-ingress --ingress-name akka-saga \
     "akka-saga:0.1.0" | kubectl apply -f -
9. If using Postman or similar, edit your etc/hosts file and add '192.168.99.100	akka-saga.io'
10. You can test this in Postman with a POST to http://akka-saga.io/bank-accounts.
Set the body to: {"customerId": "customer1", "accountNumber": "accountNumber1"}
Add the Content-Type header with a value of application/json
You should see: CreateBankAccount accepted with number: accountNumber1