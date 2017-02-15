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



























