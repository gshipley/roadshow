#** Lab 7: Adding a Database**

Most useful applications are "stateful" or "dynamic" in some way, and this is
usually achieved with a database or other data storage. In this next lab we are
going to add MongoDB to our *userXX-mlbparks* project and then rewire our
application to talk to the database using environment variables.

We are going to use the MongoDB Docker image produced by the OpenShift team -
https://github.com/openshift/mongodb

By default, this will use *EmptyDir* for data storage, which means if the Pod
disappears the data does as well. In a real application you would use
OpenShift's persistent volume mechanism with the database Pods to give them a
place to store their data. 

OpenShift is shipped with a number of application runtime and database templates
that make things like this easy. We will cover templates at the end of the lab.

###** Environment Variables **
When we use the *new-app* command this time, we need to pass in some environment
variables to be used inside the container. These environment variables are
required to set the username, password, and name of the database. You can change
the values of these environment variables to anything you would like.  The
variables we are going to be setting are as follows:

- MONGODB_USER
- MONGODB_PASSWORD
- MONGODB_DATABASE
- MONGODB_ADMIN_PASSWORD

By setting these variables when creating the Mongo database, the image will
ensure that:

- A database exists with the specified name
- A user exists with the specified name
- The user can access the specified database with the specified password

Issue the following command:

	$ oc new-app mongodb -e MONGODB_USER=mlbparks -e MONGODB_PASSWORD=mlbparks -e MONGODB_DATABASE=mlbparks -e MONGODB_ADMIN_PASSWORD=mlbparks

Again you can use the web console or the oc command to watch the progress of this command.

	$ oc get pods --watch

###**Wiring the JBoss EAP pod(s) to communicate with our MongoDB database**

When we initially created our JBoss EAP application, we provided no environment
variables. The application is looking for a database, but can't find one, and it
fails gracefully (you don't see an error).

In order for our JBoss EAP Pod(s) to be able to connect to and use the MongoDB
Pod that we just created, we need to wire them together by providing values for
the environment variables to the EAP Pod(s).  In order to do this, we simply
need to modify the DeploymentConfiguration.

First, find the name of the DC:

	$ oc get dc

Then, use the *oc env* command to set environment variables directly on the DC:

	$ oc env dc openshift3mlbparks -e MONGODB_USER=mlbparks -e MONGODB_PASSWORD=mlbparks -e MONGODB_DATABASE=mlbparks

After you have modified the DeploymentConfig object, you can verify the environment variables have been added by viewing the JSON document of the configuration:

	$ oc get dc openshift3mlbparks -o json

You should see the following section:

	env": [
		{
			"name": "MONGODB_USER",
			"value": "mlbparks"
		},
		{
			"name": "MONGODB_PASSWORD",
			"value": "mlbparks"
		},
		{
			"name": "MONGODB_DATABASE",
			"value": "mlbparks"
		}
	],

###** OpenShift Magic **
As soon as we set the environment variables on the DeploymentConfiguration, some
magic happened. OpenShift decided that this was a significant enough change to
warrant updating the internal version number of the DeploymentConfiguration. You
can verify this by looking at the output of *oc get dc*:

    ...
    openshift3mlbparks   ConfigChange, ImageChange   2

Something that increments the version of a DeploymentConfiguration, by default,
causes a new deployment. You can verify this by looking at the output of *oc get
rc*:

    openshift3mlbparks-1   openshift3mlbparks ... 0
    openshift3mlbparks-2   openshift3mlbparks ... 1

We see that the replica count for the "-1" deployment is 0. The replica count
for the "-2" deployment is 1. This means that OpenShift has gracefully torn down
our "old" application and stood up a "new" instance.

If you refresh your application:

    http://openshift3mlbparks.userXX-mlbparks.cloudapps.chicago.openshift3roadshow.com/

You'll notice that the ballparks suddenly are showing up. That's really cool!

You are probably wondering how this magically started working?  When deploying
applications to OpenShift, it is always best to use environment variables to
define connections to dependent systems.  This allows for application
portability across different environments.  The source file that performs the
connection as well as creates the database schema can be viewed here:

[DBConnection.java](https://github.com/gshipley/openshift3mlbparks/blob/master/src/main/java/org/openshift/mlbparks/mongo/DBConnection.java)

In short summary: By referring to environment variables to connect to services
(like databases), it can be trivial to promote applications throughout different
lifecycle environments on OpenShift without having to modify application code.

You can learn more about environment variables in the [environment
variables](https://docs.openshift.com/enterprise/3.0/dev_guide/environment_variables.html)
section of the Developer Guide.

###**Using the Mongo command line shell in the container **

To interact with our database we will use the *oc exec* command, which allows us
to run arbitrary commands in our Pods. In this example we are going to use the
bash shell and then invoke the mongo command while passing in the credentials
needed to authenticate to the database. First, find the name of your MongoDB
Pod:

    $ oc get pods
    NAME                         READY     STATUS       RESTARTS   AGE
    mongodb-24-rhel7-1-73af9     1/1       Running      0          13m
    openshift3mlbparks-1-build   0/1       ExitCode:0   0          36m
    openshift3mlbparks-2-qc9ug   1/1       Running      0          8m

    $ oc exec -tip mongodb-24-rhel7-1-73af9  -- bash -c 'mongo -u mlbparks -p mlbparks mlbparks'

**Note:** If you used different credentials when you created your MongoDB Pod, ensure that you substitute them for the values above.
**Note:** You will need to substitute the correct name for your MongoDB Pod.
**Note:** For windows users, you will need to use a double quote for the above command to work: oc exec -tip mongodb-24-rhel7-1-73af9  -- bash -c "mongo -u mlbparks -p mlbparks mlbparks"

Once you are connected to the database, run the following command to count the number of MLB teams added to the database:

	> db.teams.count();

You can also view the json documents with the following command:

	> db.teams.find();

**End of Lab 7**
