Persistent Storage
------------------

There are two main concepts to be aware of regarding persistent storage
in OpenShift:

* A `persistent volume` is the actual allocation of storage in OpenShift.
  These are created by the cluster administrator and will define behavior
  such as capacity, access modes, reclamation policy, and the type of
  storage (NFS, GlusterFS, AWS Elastic Block Stores, etc.)
* A `persistent volume claim` is the assignment and usage of a persistent
  volume by a user. A claim is made within the scope of a project and can
  be attached to a :term:`deployment configuration` (in practice, this
  effectively attaches it to all of the pods deployed by that configuration).

The creation of the persistent volumes is outside the scope of this guide
(by default, the CDK installation provides a few volumes for testing purposes).
This section will cover the creation and usage of the claims. Detailed
information on configuring persistent volumes can be found in the
`online documentation <https://docs.openshift.com/container-platform/latest/architecture/additional_concepts/storage.html>`_.

Persistent Volumes
~~~~~~~~~~~~~~~~~~

The first thing to keep in mind is that non-cluster admins do not have
permissions to view the configured persisted volumes. This is considered
an administrative task; users only needs to be concerned with their requested
and assigned claims::

  $ oc whoami
  user

  $ oc get pv
  User "user" cannot list all persistentvolumes in the cluster

As a reference, the same command, when run as the cluster admin, displays
details on the the volumes available::

  $ oc whoami
  admin

  $ oc get pv
  NAME      CAPACITY   ACCESSMODES   STATUS      CLAIM     REASON    AGE
  pv01      10Gi       RWO,RWX       Available                       39d
  pv02      10Gi       RWO,RWX       Available                       39d
  pv03      10Gi       RWO,RWX       Available                       39d
  pv04      10Gi       RWO,RWX       Available                       39d
  pv05      10Gi       RWO,RWX       Available                       39d

Creating a Persistent Volume Claim
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For context, the following example will be run against an existing project
with a previously deployed application. The actual behavior of the application
is irrelevant; this example will look directly at the filesystem across
multiple pods::

  $ oc project
  Using project "python-web" on server "https://10.2.2.2:8443".

  $ oc get services
  NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  python-web   172.30.29.153   <none>        8080/TCP   40m

  $ oc get dc
  NAME         REVISION   DESIRED   CURRENT   TRIGGERED BY
  python-web   2          1         1         config,image(python-web:latest)

The last line above is important to note. Volumes are not allocated directly
against a pod (or pods) but rather the deployment configuration responsible
for their creation.

Before the volume is created, a quick sanity check
:ref:`run directly on the pod <remote_shell>` shows that the mount point
does not already exist::

  $ oc get pods
  NAME                 READY     STATUS      RESTARTS   AGE
  python-web-3-sox29   1/1       Running     0          17m

  $ oc rsh python-web-3-sox29
  sh-4.2$ ls /demo
  ls: cannot access /d: No such file or directory

The ``volume`` command can be used to both request a volume and specify its
attachment point into the running containers::

  $ oc volume \
      dc python-web \
      --add \
      --claim-size 512M \
      --mount-path /demo \
      --name demo-vol
  persistentvolumeclaims/pvc-axv7b
  deploymentconfigs/python-web

There is quite a bit going on in that command, so a breakdown of the pieces
is warranted:

* ``dc python-web`` indicated where the volume will be added. As with other
  many other commands in the client, two pieces of information are needed:
  the resource type ("dc" is shorthand for "deployment configuration")
  and the name of the resource. Alternatively, they could be joined with
  a front slash into a single term (``dc/python-web``).
* ``--add`` is the volume action being performed. In this case, a volume is
  being added to the configuration. By comparison, ``--remove`` is used to
  detach a volume.
* ``--claim-size 512M`` requests a volume `of at least 512MB`. The actual
  allocated volume may be larger as OpenShift will fulfill the claim with
  the best fit available.
* ``--mount-path /demo`` dictates where in the pod(s) to mount the storage
  volume.
* ``--name demo-vol`` is an identifier name used to reference the volume
  at a later time.

The output shows that a new claim has been created (``pvc-axv7b``) and the
deployment configuration ``python-web`` has been edited.

There are a few things to verify at this point. Details about a claim can
be found under the ``pvc`` resource type::

  $ oc get pvc
  NAME        STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
  pvc-axv7b   Bound     pv05      10Gi       RWO,RWX       40m

  $ oc describe pvc pvc-axv7b                                                                                                                                                        1 ↵
  Name:          pvc-axv7b
  Namespace:     python-web
  Status:        Bound
  Volume:        pv05
  Labels:        <none>
  Capacity:      10Gi
  Access Modes:  RWO,RWX
  No events.

Notice that, despite the claim only requesting 512M, the provided capacity is
10G. The output of the ``get pv`` command above shows that the installation
is only configured with 10Gi volumes, which makes it the "best fit" for the
claim.

The ``volume`` command can also provide details on the attached volume,
including which deployment configurations have volumes and where they are
mounted::

  $ oc volume dc --all
  deploymentconfigs/python-web
    pvc/pvc-axv7b (allocated 10GiB) as demo-vol
      mounted at /demo

Repeating the earlier test on the pod, there is now a ``/demo`` mount point
available::

  $ oc rsh python-web-3-sox29
  sh-4.2$ ls /demo
  sh-4.2$

Persistent Volumes Across Pods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To reinforce the concept that the volume is attached to the deployment
configuration, and thus all pods spawned by it, the application can be
scaled and used to show the mount points refer to the same volume::

  $ oc scale dc python-web --replicas 2
  deploymentconfig "python-web" scaled

  $ oc get pods
  NAME                 READY     STATUS      RESTARTS   AGE
  python-web-3-ka3y2   1/1       Running     0          1m
  python-web-3-sox29   1/1       Running     0          1h

The newly created pod, ``ka3y2``, will have the same configuration as the
previous pod since they were created from the same deployment configuration.
In particular, this includes the mounted volume::

  $ oc rsh python-web-3-ka3y2
  sh-4.2$ ls /demo
  sh-4.2$

Proof that they refer to the same volume can be seen by adding a file to the
volume on one of the pods and verifying its existence on the other::

  $ oc rsh python-web-3-ka3y2
  sh-4.2$ echo "Hello World" > /demo/test
  sh-4.2$ exit

  $ oc rsh python-web-3-sox29
  sh-4.2$ ls /demo
  test
  sh-4.2$ cat /demo/test
  Hello World
  sh-4.2$

Detaching a Persistent Volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In addition to demonstrating the volume is shared across pods, it is also
important to emphasize the "persistent" aspect of it. The ``volume`` command
is also used to detach a volume from a configuration (and thus all of its
pods)::

  $ oc volume dc python-web --remove --name demo-vol
  deploymentconfigs/python-web

Note that the ``--name demo-vol`` argument refers to the name specified
during creation above.

Attempting to reconnect to the pod to verify the volume was detached shows
a potentially surprising result::

  $ oc rsh python-web-3-sox29                                                                                                                                                        1 ↵
  Error from server: pods "python-web-3-sox29" not found

Keep in mind that changing a pod's volumes is no different than any other
change to the configuration. The pods are not updated, but rather the old
pods are scaled down while new pods, with the updated configuration, are
deployed. The deployment configuration event log shows that a new deployment
was created for the change (the format of the output below is heavily modified
for readability, but the data was returned from the call)::

  $ oc describe dc python-web
  Events:
    FirstSeen  Reason             Message
    ---------  ------             -------
    1h         DeploymentCreated  Created new deployment "python-web-2" for version 2
    1h         DeploymentCreated  Created new deployment "python-web-3" for version 3
    11m        DeploymentScaled   Scaled deployment "python-web-3" from 1 to 2
    2m         DeploymentCreated  Created new deployment "python-web-4" for version 4

The four events listed correspond to the examples run in this section:

#. Initial successful deployment (the "version 1" intentionally skipped).
#. The "version 3" deployment corresponds to adding the volume.
#. The scaling operation retains the deployment version (the "3" in
   ``python-web-3``) and creates a new pod.
#. The "version 4" deployment was made to activate the change to remove the
   volume.

Getting back to verifying the volume was removed, one of the new pods can be
accessed to check for the presence of the mount point::

  $ oc get pods
  NAME                 READY     STATUS      RESTARTS   AGE
  python-web-4-j3u0r   1/1       Running     0          11m
  python-web-4-vrq2t   1/1       Running     0          11m

  $ oc rsh python-web-4-j3u0r
  sh-4.2$ ls /demo
  ls: cannot access /demo: No such file or directory

Reattaching a Persistent Volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Detaching a volume from a deployment configuration does not release the
volume or reclaim its space. Listing the volumes as above (again as a
cluster admin) shows the volume is still in use::

  $ oc get pv
  NAME      CAPACITY   ACCESSMODES   STATUS      CLAIM                  REASON    AGE
  pv01      10Gi       RWO,RWX       Available                                    39d
  pv02      10Gi       RWO,RWX       Available                                    39d
  pv03      10Gi       RWO,RWX       Available                                    39d
  pv04      10Gi       RWO,RWX       Available                                    39d
  pv05      10Gi       RWO,RWX       Bound       python-web/pvc-axv7b             39d

Additionally, as the non-cluster admin user, the claim is still present as
a resource::

  $ oc get pvc
  NAME        STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
  pvc-axv7b   Bound     pv05      10Gi       RWO,RWX       1h

The volume can be added back into the deployment configuration (or a different
one if so desired) using a variation of the ``volume`` command initially used::

  $ oc volume \
      dc python-web
      --add
      --type pvc
      --claim-name pvc-axv7b
      --mount-path /demo-2
      --name demo-vol-2
  deploymentconfigs/python-web

The difference in this call is that instead of specifying details about the
claim being requested (such as its capacity), a specific claim is referenced
(the name being found using the ``get pvc`` command above). For demo purposes,
it has been mounted to a slightly different path and using a different volume
name.

As with the previous configuration changes, new pods have been deployed.
Connecting to one of these pods shows the contents of the volume were
untouched::

  $ oc get pods
  NAME                 READY     STATUS      RESTARTS   AGE
  python-web-5-49tsa   1/1       Running     0          2m
  python-web-5-s8yni   1/1       Running     0          2m

  $ oc rsh python-web-5-49tsa
  sh-4.2$ ls /demo-2
  test
  sh-4.2$ cat /demo-2/test
  Hello World
  sh-4.2$

Releasing a Persistent Volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The example above demonstrates that removing a volume does not release the
volume nor delete its contents. That requires another step. Remember that
claims are resources, similar to routes or deployment configurations, and
can be deleted in the same fashion (using the updated name from the reattach
example)::

  $ oc volume dc python-web --remove --name demo-vol-2
  deploymentconfigs/python-web

  $ oc delete pvc pvc-axv7b
  persistentvolumeclaim "pvc-axv7b" deleted

Listing the claims for the user shows none allocated::

  $ oc get pvc

The cluster administrator shows that the previously bound volume is now free::

  $ oc get pv
  NAME      CAPACITY   ACCESSMODES   STATUS      CLAIM     REASON    AGE
  pv01      10Gi       RWO,RWX       Available                       39d
  pv02      10Gi       RWO,RWX       Available                       39d
  pv03      10Gi       RWO,RWX       Available                       39d
  pv04      10Gi       RWO,RWX       Available                       39d
  pv05      10Gi       RWO,RWX       Available                       39d

