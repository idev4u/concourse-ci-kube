# The missing kubernetes deployment for Concourse-Ci

## Goal
This project was build to bring Concourse-Ci to IBM Cloud Container Service, which is based on kubernetes.
If someone else has the need to deploy Concourse-Ci to kubernetes he/she should also benefit from this project.

## Pre Requirements
The only Requirements to follow all this steps is to have running kubernetes cluster on IBM Bluemix Containers.
How to do this, is described on https://console.bluemix.net/containers-kubernetes/launch?env_id=ibm:yp:eu-de
The second requirement is that you have basic knowledge about kubernetes.

## Install

1. Grab the project

	```shell
	git clone git@github.com:idev4u/concourse-ci-kube.git
	cd concourse-ci-kube
	```
2. As described in the [Concourse Documentation](http://concourse.ci/binaries.html) generate the keys for TSA and ATC

	```
	mkdir -p keys/web keys/worker

	ssh-keygen -t rsa -f ./keys/web/tsa_host_key -N ''
	ssh-keygen -t rsa -f ./keys/web/session_signing_key -N ''

	ssh-keygen -t rsa -f ./keys/worker/worker_key -N ''

	cp ./keys/worker/worker_key.pub ./keys/web/authorized_worker_keys
	cp ./keys/web/tsa_host_key.pub ./keys/worker
	```

3. Generate the secrets volume for your deployments

	This generate the secret volume for the *web* deployment
	```
	$ kubectl create secret generic concourse-web-keys \
		--from-file=./keys/web/authorized_worker_keys \
		--from-file=./keys/web/session_signing_key \
		--from-file=./keys/web/session_signing_key.pub \
		--from-file=./keys/web/tsa_host_key \
		--from-file=./keys/web/tsa_host_key.pub
	```
	If you would verify the generated secrets volume, use this command.  
	```shell
	kubectl get secret concourse-web-keys -o yaml
	```

	Here is command that generate the secret volume for the *worker* deployment
	```
	$ kubectl create secret generic concourse-worker-keys \
		--from-file=./keys/worker/tsa_host_key.pub \
		--from-file=./keys/worker/worker_key \
		--from-file=./keys/worker/worker_key.pub
	```
	And here as well the verify command.  
	```shell
	kubectl get secret concourse-worker-keys -o yaml
	```

4. Deploy the all the 3 components

	This command will deploy the database for your Concourse-Ci  
	```shell
	kubectl apply -f concourse-db-deployment.yaml && kubectl apply -f concourse-db-service.yaml
	```

	```console
	deployment "concourse-db" created
	service "concourse-db" created
	```
	Before you deploy the Web UI, change the value of the external IP in the *concourse-web-deployment.yaml* file

	```
	  - name: CONCOURSE_EXTERNAL_URL
	    value: ${kubernetes_node_public_ip}
	```
	This command will deploy the WebUI of your Concourse-Ci  
	```shell
	kubectl apply -f concourse-web-deployment.yaml && kubectl apply -f concourse-web-service.yaml
	```

	```console
	deployment "concourse-web" created
	service "concourse-web" created
	```
	This command will deploy one worker for your Concourse-Ci  
	```
	bash$ kubectl apply -f concourse-worker-deployment.yaml && kubectl apply -f concourse-worker-service.yaml
	```

	```console
	deployment "concourse-worker" created
	service "concourse-worker" created
	```

## Concourse Pipeline

After the Concourse-Ci deployment is succesfully done, you can login the frist time into Concourse-Ci. Open the url `http://${kubernetes_node_public_ip}:32080` in your favorite browser and login with user __concourse__ and the password  __changeme__. If you have changed this values in the deployment manifests, use yours. If this works and you have download the tool fly you can push your first pipeline.

```shell
fly -t kube login -c ${kubernetes_node_public_ip}
fly -t kube set-pipeline -p kube-pipe -c pipeline/pipeline.yml
fly -t kube expose-pipeline -p kube-pipe
```

## Some useful command to be succesful

### public IP of your node
How did you find the public IP of your kubernetes node on the Bluemix Container platform? This is the command for getting this information.

```sh
bx cs workers concourse-ci
```
```console
ID                                                 Public IP      Private IP      Machine Type   State    Status
kube-par01-pa747d8ee7d506411aba3f992fc3d3c7a1-w1   x.x.x.x        10.x.x.x	  free           normal   Ready
```
### kube context

If you are not sure, that you aim on the correct context, this command helps you.
```shell
kubectl config current-context
```
```console
concourse-ci
```

This command provides an overview of your pods you should have 3!
```shell
kubectl get pods -o wide
```
```console
NAME                                READY     STATUS    RESTARTS   AGE       IP               NODE
concourse-db-59888000-1fcv0         1/1       Running   0          6h      172.x.x.x      10.x.x.a
concourse-web-2821356835-npkvb      1/1       Running   0          5h      172.x.x.x      10.x.x.a
concourse-worker-1074565060-nkrm9   1/1       Running   0          13m     172.x.x.x      10.x.x.a
```

This command provides an overview of your service you should also have 3!
```shell
bash$ kubectl get svc -o wide
```
```console
NAME               CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE       SELECTOR
concourse-db       None         <none>        55555/TCP        28d       service=concourse-db
concourse-web      None         <nodes>       8080:32080/TCP   10m       service=concourse-web
concourse-worker   None         <none>        55555/TCP        12m       service=concourse-worker
```

## Troubleshooting

### TSA Connection
*Problemdescription:*

I had a problems with the tsa connection from inside of the worker container.

*Solution:*  
I fixed it with
replaceing the service selector name with the endpoint ip.

Getting the enpoint of the service `concourse-web` 

```
kubectl get endpoints | grep web
concourse-web      1.1.1.80:8080    17h
```
and here the area which is have to change concourse-worker-deployment.yaml

```yml
   ...
   # use the endpoint ip, because the dns lookup point to the cluster ip and this ip is not reachable from inside the container
   # value: concourse-web
   value: 1.1.1.80
   ...
```
