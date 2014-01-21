The example shown here models a hypothetical application called
"Guitars", a simple web-based service using several functional roles
like web, app and database services.

The purpose of this example is to show how dev and ops teams can
collaborate around how to restart the web tier, manage software
promotions, run status health checks and a nightly batch job.

This is a single-machine vagrant configuration that installs
and configures a rundeck instance and an Apache httpd instance.

The httpd instance is used as a simple web-based file
repository from which scripts and job options are shared to Rundeck. 


To run this example, you will bring up a VM using vagrant, log in 
to Rundeck and perform certain jobs.

## Vagrant configuration

This vagrant configuration defines one virtual machine:

* **rundeck**: The rundeck instance used by all the teams.

The rundeck VM runs a centos base box and installs software via yum/rpm.

If you are curious how the rundeck and apache instances are installed see
the vagrant provisioning scripts:

* [install-rundeck.sh](https://github.com/rundeck/guitars-demo/blob/master/install-rundeck.sh): Installs java, rundeck and the hipchat notification plugin
along with some utility packages like xmlstarlet.
* [add-project.sh](https://github.com/rundeck/guitars-demo/blob/master/add-project.sh): Creates the "guitars" rundeck project and installs the jobs, configures the user accounts,
nodes, ssh access, and copies scripts to the apache document root.
* [install-httpd.sh](https://github.com/rundeck/guitars-demo/blob/master/install-httpd.sh): Installs Apache httpd, creates the document root for scripts and options and enables the mod_dav plugin to provide future "PUT"-based access to publish files.

### Requirements

* Internet access to download packages from public repositories.
* [Vagrant 1.2.2](http://downloads.vagrantup.com)

### Startup

Start up the VM like so:

    vagrant up rundeck

You can access the rundeck and httpd instances from your host machine using the following URLs:

* rundeck: http://192.168.50.2:4440
* httpd: http://192.168.50.2/guitars

## User stories

Every demo needs a story. There are 3 stories in this demo. The unifying theme to all of them is "low MTTB" (with Rundeck you have the lowest "Mean Time to Button") for self service
and increasing the level of visibility for everybody.

### Story #1

* Actors: Developers, Release Engineering, Operations

When an Guitars web server acts up under load, it needs to be restarted. At this point, someone from the NOC Team calls the Release Engineering team to do the restart. But under load, the Guitars web servers don't always stop using the normal method. When this happens, the Devs need to get involved to run their special script to forcibly stop the web server processes. Not only is this an inefficient process, but due to compliance reasons, the ops team can't give shell login access to the Devs to view logs and check the web servers process status.

So the team turns to Rundeck. The Dev provided scripts are plugged into Rundeck jobs that can be safely and securely called by the NOC (and both receive notifications and know how to follow the output). Once the devs are happy with how the procedure works, they can handoff a Restart button to the NOC Team which allows ops a safe and secure way to call the required restart method themselves. A "Status" job is runnable by both Developers and NOC teams to check on the health of the web servers at any time.


### Story #2

* Actors: Release Engineering

The Release Engineering team needs a method to promote new versions of the Guitars software to the artifact repositories used by Operations to do deployments. Because there are several upstream repositories (eg, CI, stable and release) containing any number of releases and associated package versions, the Job should contain smart menus to let users drill down to the package versions they want to promote. We want a mistake-proof method of executing the promotion, and we want it to be logged and visible to all in our organization. 

So the team turns to Rundeck. The promotion scripts that pull from one repository and upload to another are plugged into Rundeck jobs. Using Rundeck option providers, the jobs are able to have drop down menus that are populated with the correct repositories and their available artifacts. 


### Story #3

* Actors: Developers, Operations

A nightly batch job needs to be run to rebuild catalog data. The developers have written a series of procedures to do the catalog rebuild and they need Operations to run them. The job needs to run at a regularly scheduled time and the right people need to be notified if there is a failure. This is a critical business process, so everyone in the organization needs a known place to look to see the status of this job.

So the team turns to Rundeck. The scripts are plugged into Rundeck jobs. The catalog rebuild job is now run automatically by Rundeck each night and the appropriate notifications are sent. Also, the Ops team has given business managers the ability to run the job on demand if they need an update sooner than the next scheduled nightly run.

Also, because the ops team uses [HipChat](http://www.hipchat.com) for a shared running chat log, notifications
should also be configured to send job status there.

## Logins and access control

The rundeck instance is configured with several logins (user/password),
each with specialized roles. Roles are allowed to see and perform only
the jobs they are granted access to use.

* admin/admin: The "admin" login has full privileges and create, edit, run jobs and ad-hoc commands.
* ops/ops: The "ops" login is able to run jobs but not create or modify them.
* dev/dev: The "dev" login is able to run the "Status" job and look at all logs.
* releng/releng: The "releng" login is able to run the "Promote" job and look at all logs.

After logging in as any of the users mentioned above, click on the user's profile page 
to see a list of that users group memberships.


The [guitars.aclpolicy](https://github.com/rundeck/guitars-demo/blob/master/guitars.aclpolicy)
file specifies what actions users like the "ops" and "dev" can do. All groups can 
view information about the nodes, jobs, and history so everybody has basic visibility.

When logging into each of the users, notice how the job listing and job toolbar reflect
the permissions of each user.

## Nodes

The guitars project contains several nodes. Go to the "Run" page and press the button
"Show all nodes". You will see the following nodes:

* app_1
* app_2
* db_1
* www_1
* www_2

Each of the nodes play a role in running the Guitars application.

The Tags columns shows how each node is tagged with user defined
labels. You can use tags for grouping or classification.
For example, all of the nodes tagged "guitars" represent the
guitars hosts.
There are also tags that describe functionally like "www" and "app" and "db".
Clicking on any of the tag names filters the nodes for ones that are tagged with that label.
Clicking on the "guitars" tag will list all the guitars nodes again.


Pressing the the node name reveals the node's metadata. A node can have any number
of user defined attributes but some "standard" info is included like OS Family,Name,Architecture.
You will also see some metadata specific to Guitars is also shown like "guitars-customer" and "guitars-location". Rundeck node metadata is accessible to any command, script or 
rundeck job to help you keep them environment independent. 
Here's the metadata for the "www_1.guitars.com" node:

    www_1.guitars.com:
        osFamily: unix
        tags: guitars, www
        username: www_1
        osArch: x86_64
        osVersion: 2.6.32-279.el6.x86_64
        description: A www server node.
        nodename: www_1.guitars.com
        hostname: localhost
        guitars-location: US-East
        guitars-customer: acme.com
        ssh-keypath: /var/lib/rundeck/.ssh/id_rsa
        osName: Linux

### Making one node look like six

Since this is a single-machine Vagrant instance,
a little bit of cleverness is used to make the single node look like it is actually six.
Each node is is uniquely named (eg www_1.guitars.com) and given its own Linux system account (eg www_1). Each node shares the same hostname (eg localhost, the rundeck server). The Rundeck server
ssh's to the appropriate node's username to perform any needed action by the rundeck Jobs.
This is equivelent to saying `rundeck-server $ ssh www_1@localhost <command>`.

While this example makes use of the bundled SSH support, Rundeck command dispatching is 
completely pluggable and open ended to use your desired execution framework (eg, winrm, salt, mcollective, custom-xml-rpc, etc).

You can retrieve the node info for a project using the Rundeck Web API.
This URL lists the resources for guitars: 
http://192.168.50.2:4440/api/5/project/guitars/resources .
Of course, this is canned demo data and a real rundeck project generally gets
this node info from an external source like your CMDB, Chef, Puppet, AWS, Rightscale etc.

### Jobs

The rundeck instance will come up with the following demo jobs 
already loaded. All jobs are organized under a common group called "guitars".

- guitars/Promote - 'promote the packages'
- guitars/web/Restart - 'restart the web servers'
- guitars/Status - 'Check the status of guitars'
- guitars/nightly_catalog_rebuild - 'rebuild the catalog data'
- guitars/web/start - 'start the web servers'
- guitars/web/stop - 'stop the web servers'

Each job is defined in its own file using the 
[XML format](http://rundeck.org/docs/manpages/man5/job-v20.html). 
[YAML](http://rundeck.org/docs/manpages/man5/job-yaml-v12.html) could also have been used as an alternative syntax. Rundeck jobs can call
scripts stored in a web server by specifying its location with a 
[scripturl](http://rundeck.org/docs/manpages/man5/job-v20.html#script-sequence-step).
Storing scripts outside of a rundeck job, faciliates better collaboration
and configuration management.

Using job groups is optional but is often helpful to organize procedures
and simplify setting up ACL policies.

#### Promote

The Promote job is run by the Release Engeering team to publish artifacts from
upstream repositories to ones used by operations.

A key part to the promote job is a user interface that lets users manage a hierarchical set of job choices.
The Promote job prompts the user for several choices about which package versions to publish
in the ops package repo. 
The `option` elements specified in the Promote job definition read choices from the 
[valuesUrl](http://rundeck.org/docs/manpages/man5/job-v20.html#valuesurl-json), which returns JSON data consumable by rundeck. This JSON can be static
files like in this example, but more typically is generated by an external service or tool.

The Promote job contains a trivial script which simply prints out the job runner's choices
but does show how a script can access options data set by the job.

* [job source](https://github.com/rundeck/guitars-demo/blob/master/jobs/Promote.xml)

#### Restart

The Restart job is run by the Operations team to manage the web tier's restart procedure.

The Restart job includes a "method" option to support the two methods to stop the web servers, "normal" and "force".
Also, because the location of the application installation directory is expected to
vary, a "dir" option is also presented. 

This job is defined to execute on nodes tagged "www". The Restart job actually builds on two lower level jobs, web/stop and web/start. This kind of job composition is typical for rundeck users as it gives them building blocks to create higher levels of automation later.

* [job source](https://github.com/rundeck/guitars-demo/blob/master/jobs/Restart.xml)

#### Status

The Status job is used by both Developers and Operations to get a heath check of the
web tier.

The Status job executes a procedure written by the devs to check the health of the web tier.
By default, rundeck jobs stop if a failure occurs. This Status job takes advantage of rundeck
[step error handlers](http://rundeck.org/docs/manual/job-workflows.html#error-handlers) to continue going to the next node if the status check fails.

This job is defined to execute on nodes tagged "www".

* [job source](https://github.com/rundeck/guitars-demo/blob/master/jobs/Status.xml)

#### nightly_catalog_rebuild

The nightly_catalog_rebuild job is provided by developers to run automatically every night.

The nightly_catalog_rebuild job is meant to run at 00:00:00 (midnight) every day.
The [schedule](http://rundeck.org/docs/manpages/man5/job-v20.html#schedule) element in the job definition specifies this in a crontab like format.
Also, the [notification](http://rundeck.org/docs/manpages/man5/job-v20.html#notification) element is used to send emails upon success and failure to the 
"bizops@guitars.com" mail group.

The script for this job runs on the database server tagged "db". The script
is written to use 
[context variables](http://rundeck.org/docs/manual/job-workflows.html#context-variables)
which exposes metadata about the job and node information from the resource model. 
Attributes about the node
are exposed as environment variables to the script. For example, the db
node has two custom attributes: guitars-customer and guitars-location. The
values of these metadata attributes can be read by the script as environment variables:
`RD_NODE_ANVILS_CUSTOMER` and `RD_NODE_ANVILS_LOCATION`.

* [job source](https://github.com/rundeck/guitars-demo/blob/master/jobs/nightly_catalog_rebuild.xml)

## Where to go from here

This demo helps introduce new users to Rundeck and gives an idea for how Rundeck can help
with handoffs between teams, increase visibility and provide self service. 

See the [Documentation](http://rundeck.org/docs.html) for more information.

