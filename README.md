# Building a restful Web service using Kubernetes (K8s)

## Video 1

This service will have some core characteristics.

- The application will run in the Azure container service as a K8s cluster
- It will host a web service (Python + Flask)
- It will include a caching layer (Redis)
- It will have a backend database of MySQL
- It will run in two pods
	- the first pod will have both the web service and the caching layer
	- the second pod will have a MySQL database

The prerequisite to this session is the fact that you have a Kubernetes cluster already provisioned.

That lab can be found here: https://www.youtube.com/watch?v=GnMdiTuU_lI&t=94s

#### Downloading the source code

You can download all the source code here:

https://github.com/brunoterkaly/k8s-multi-container

The command download is:

	git clone https://github.com/brunoterkaly/k8s-multi-container.git

You will notice that there are three folders:

- Build (contains the Python code to build out our web service)
- Deploy (Contains the YML definition file that lets us set up our pods, services, and our containers in Kubernetes)
- Run (contains the commands to deploy, operate, and manage our Kubernetes cluster)

## Video 2

In this second video we are building out the web service and focusing on the Python code. We will also build out the docker image that will run in the agents of the Kubernetes cluster. We will review the Python file, Flask and some other pieces.

- The web service is a combination of Python and Flask
- Python represents the main programming paradigm and flask as the necessary libraries to expose endpoints as web services
- dockerfile is used to build out our image, which will then be uploaded to hub.docker.com
- app.py represents the Python application that implements web services
- buildpush.sh is some script code to do both the building of the image, as well as the uploading of the image to hub.docker.com


#### Code summary




#### dockerfile

- Represents the image blueprint for our docker container
- Listening on port 5000
- Starts up running app.py

Dockerfile

	FROM python:2.7-onbuild
	EXPOSE 5000
	CMD [ "python", "app.py" ]


#### app.py

- The web service application
- Imports a variety of libraries

Expose the "init" endpoint
- The user will hit http://ourwebservice/init
- And all this database code will execute
- 

Expose the "add" endpoint
- The user will hit http://ourwebservice/courses/add
- Performs an insert into the database using the data posted by the user
- POST is the HTTP verb being used


Expose the "courses/{course id}" endpoint
- The user will hit http://ourwebservice/courses/{course id}
- This allows the user to query the endpoint
- It will return records from either the cache or the database
- If the data is not in the cache, it will be retrieved from the database, put into the cache, and returned to the client

app.py

	from flask import Flask
	from flask import Response
	from flask import request
	from redis import Redis
	from datetime import datetime
	import MySQLdb
	import sys
	import redis 
	import time
	import hashlib
	import os
	import json
	
	app = Flask(__name__)
	startTime = datetime.now()
	R_SERVER = redis.Redis(host=os.environ.get('REDIS_HOST', 'redis'), port=6379)
	db = MySQLdb.connect("mysql","root","password")
	cursor = db.cursor()
	
	@app.route('/init')
	def init():
	    cursor.execute("DROP DATABASE IF EXISTS AZUREDB")
	    cursor.execute("CREATE DATABASE AZUREDB")
	    cursor.execute("USE AZUREDB")
	    sql = """CREATE TABLE courses(id INT, coursenumber varchar(48), 
	                  coursetitle varchar(256), notes varchar(256));  
	     """
	    cursor.execute(sql)
	    db.commit()
	    return "DB Initialization done\n" 
	
	@app.route("/courses/add", methods=['POST'])
	def add_courses():
	
	    req_json = request.get_json()   
	    cursor.execute("INSERT INTO courses (id, coursenumber, coursetitle, notes) VALUES (%s,%s,%s,%s)", 
	          (req_json['uid'], req_json['coursenumber'], req_json['coursetitle'], req_json['notes']))
	    db.commit()
	    return Response("Added", status=200, mimetype='application/json')
	
	@app.route('/courses/<uid>')
	def get_courses(uid):
	    hash = hashlib.sha224(str(uid)).hexdigest()
	    key = "sql_cache:" + hash
	    
	    returnval = ""
	    if (R_SERVER.get(key)):
	        return R_SERVER.get(key) + "(from cache)" 
	    else:
	        cursor.execute("select coursenumber, coursetitle, notes from courses where ID=" + str(uid))
	        data = cursor.fetchall()
	        if data:
	            R_SERVER.set(key,data)
	            R_SERVER.expire(key, 36)
	            return R_SERVER.get(key) + "\nSuccess\n"
	        else:
	            return "Record not found"
	
	if __name__ == "__main__":
	    app.run(host="0.0.0.0", port=5000, debug=True)


#### buildpush.sh

buildpush.sh

	docker build . -t brunoterkaly/py-red
	docker push brunoterkaly/py-red


## Video 3

In this third video we will start looking at some YML files.

- These files define the way our containers will run in the Kubernetes cluster
- They also define how the containers are organized within pods
- Pods group together containers in the same host
- This provides efficiency in that the containers are running close together
- As described, there are two pods
- On the first pod there will be two containers, the Python web service and the Redis Cache
- On the second pod there will be the MySQL database
- YML files just text file that define the pods and the services
- The first part is called the 'web' pod 


<table style="float: none; border-bottom: #b4b4b4 1px solid; text-align: left; border-left : #b4b4b4 1px solid; border-collapse: collapse; 
              font-family: Calibri, 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #000000; border-top: medium none; 
			  border-right: #b4b4b4 1px solid;"><tr style="background-color:rgb(242,247,247); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		run.sh	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		A custom script to execute all the necessary commands for the deployment</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		script file</td></tr><tr style="background-color: rgb(216,223,230); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		clean.sh	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		Shuts down all the pods and services in a cluster</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		script file</td></tr><tr style="background-color:rgb(242,247,247); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		db-pod.yml	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		The definition of the pod for the MySQL database</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		yml file</td></tr><tr style="background-color: rgb(216,223,230); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		db-svc.yml	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		The service layer that provides the interface to the underlying pod</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		yml file</td></tr><tr style="background-color:rgb(242,247,247); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		web-pod-1.yml	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		The definition of the pod for the Web Service and the Redis Cache database</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		yml file</td></tr><tr style="background-color: rgb(216,223,230); vertical-align: top"><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		web-svc.yml	</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		The service layer that provides the interface to the underlying pod</td><td style="padding: 5px 10px; border-left: 1px solid #b4b4b4;  border-top: 1px solid #b4b4b4;">
		yml file</td></tr></table>

#### web-pod-1.yml

Defines the pod for the web part of the application

web-pod-1.yml

	apiVersion: "v1"
	kind: Pod
	metadata:
	  name: web1
	  labels:
	    name: web
	    app: demo
	spec:
	  containers:
	    - name: redis
	      image: redis
	      ports:
	        - containerPort: 6379
	          name: redis
	          protocol: TCP
	    - name: python
	      image: brunoterkaly/py-red
	      env:       
	        - name: "REDIS_HOST"
	          value: "localhost"
	      ports:
	        - containerPort: 5000
	          name: http
	          protocol: TCP                    




#### db-pod.yml

db-pod.yml

	apiVersion: "v1"
	kind: Pod
	metadata:
	  name: mysql
	  labels:
	    name: mysql
	    app: demo
	spec:
	  containers:
	    - name: mysql
	      image: mysql:latest
	      ports:
	        - containerPort: 3306         
	          protocol: TCP
	      env: 
	        - 
	          name: "MYSQL_ROOT_PASSWORD"
	          value: "password"


#### web-svc.yml
                          
web-svc.yml
            
	apiVersion: v1
	kind: Service
	metadata:
	  name: web
	  labels:
	    name: web
	    app: demo
	spec:
	  selector:
	    name: web 
	  type: NodePort
	  ports:
	   - port: 80
	     name: http
	     targetPort: 5000
	     protocol: TCP

#### db-svc.yml

db-svc.yml

	apiVersion: v1
	kind: Service
	metadata:
	  name: mysql
	  labels:
	    name: mysql
	    app: demo  
	spec:
	  ports:
	  - port: 3306
	    name: mysql
	    targetPort: 3306
	  selector:
	    name: mysql
	    app: demo
	
### Getting the 'config' file

We will need the config file to communicate with the cluster. Getting it is pretty easy.

- We will use the Python and Azure SDK's to retrieve this file
- It will put it in a special folder from which we will copy it into the current directory

The commands

	az acs kubernetes get-credentials --resource-group=ascend-k8s-rg --name=ascend-k8s-svc
	cp ~/.kube/config .

### Setting environment variable (KUBECONFIG) and getting kubectl

This environment variable is used by kubectl to know where to retrieve the config file.

Command looks like this:

	export KUBECONFIG=`pwd`/config

We will also want to get the kubectl utility, which is the mechanism we can use to communicate with their Kubernetes cluster. It works in conjunction with the config file just addressed.

	az acs kubernetes install-cli
	kubectl cluster-info


### Running the pods and services

At this point we are going to run the pods and services.

- We will run the file 'run.sh'
- It will use the YML files
- It will create pods and services

run.sh

	kubectl create -f db-pod.yml
	kubectl create -f db-svc.yml
	kubectl create -f web-pod-1.yml
	kubectl create -f web-svc.yml



Command to list running pods:
	
	kubectl get pods

Command to list running services:

	kubectl get svc

Command to expose an external IP address for our Web service:

	kubectl edit svc/web

Editing the web service file should look like this. nnotice that we did switch the  **type ** element to **LoadBalancer**.

	# Please edit the object below. Lines beginning with a '#' will be ignored,
	# and an empty file will abort the edit. If an error occurs while saving this file will be
	# reopened with the relevant failures.
	#
	apiVersion: v1
	kind: Service
	metadata:
	  creationTimestamp: 2017-02-15T22:00:51Z
	  labels:
	    app: demo
	    name: web
	  name: web
	  namespace: default
	  resourceVersion: "149793"
	  selfLink: /api/v1/namespaces/default/services/web
	  uid: 36eee4e7-f3ca-11e6-a3cd-000d3a9203fe
	spec:
	  clusterIP: 10.0.73.141
	  ports:
	  - name: http
	    nodePort: 30189
	    port: 80
	    protocol: TCP
	    targetPort: 5000
	  selector:
	    name: web
	  sessionAffinity: None
	  type: LoadBalancer
	status:
	  loadBalancer:
	    ingress:
	    - ip: 13.89.230.209


## Testing the cluster

At this point we can start testing the cluster.  The pods and services are already running.

- In the last section we even exposed an external IP address, allowing us to  pretty much access web service from anywhere
- As you recall, there are three endpoints that we can use to access the web service

	http<nolink>://XXX/init
	http<nolink>://XXX/courses/add
	http<nolink>://XXX/courses/{course id}
	
## Video 3

### Testing the application

We will initialize the database by hitting that endpoint using the following syntax:

	curl http://13.89.230.209/init

Inserting data into the application ccan be done as follows. Be sure to remember that that public IP address gets change the public IP address that relates to your web service..

	curl -i -H "Content-Type: application/json" -X POST -d '{"uid": "1", "coursenumber" : "401" , "coursetitle" : "K8s", "notes" : "An orchestrator"}' http://13.89.230.209/courses/add
	
	curl -i -H "Content-Type: application/json" -X POST -d '{"uid": "2", "coursenumber" : "402" , "coursetitle" : "DC/OS", "notes" : "for big workloads"}' http://13.89.230.209/courses/add
	
	curl -i -H "Content-Type: application/json" -X POST -d '{"uid": "3", "coursenumber" : "403" , "coursetitle" : "Docker Swarm", "notes" : "Easy to use"}' http://13.89.230.209/courses/add
	
And finally, to actually query by UID you will issue the following commands:
	
	curl http://13.89.230.209/courses/1
	curl http://13.89.230.209/courses/2
	curl http://13.89.230.209/courses/3

And if you were to do it a second time. You would get information coming back from the cache rather than from the database itself.


	
