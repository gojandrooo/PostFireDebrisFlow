# Feature extraction on Nautilus Kubernetes Cluster

This documents describes how the [Nautilus](https://ucsd-prp.gitlab.io/) Kubernetes 
cluster is used to extract features in parallel using Dask.

## Persistent Storage
The first step consists in having access to the data from within the K8s cluster.  A 
[persistent volume](https://ucsd-prp.gitlab.io/userdocs/storage/ceph-posix/) is 
created using CephFS.  This is the [storage.yaml](storage.yaml) file:

	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
  		name: staley-debrisflow
	spec:
  		storageClassName: rook-cephfs
  		accessModes:
  		- ReadWriteMany
  		resources:
    		requests:
      		storage: 20Gi

Then create the empty volume using

	kubectl create -f storage.yaml 

Check that persistent volume claim (PVC) is running:

	kubectl get persistentvolumeclaims


Next open a new pod which mounts this volume, specified in the file 
[access_storage.yaml](access_storage.yaml), and start it using:

	kubectl create -f access_storage.yaml

Check that the pod is running:

	kubectl get pods

Ssh into this pod, and use scp to copy the data from your computer into the container's directory `/dfdata/`:

	kubectl exec -ti df-access-pod -- bash

Alternatively, you can directly copy the data into the container using kubectl:

    kubectl cp staley_debrisflow.parquet df-access-pod:/dfdata/staley_debrisflow.parquet
    kubectl cp extract_contributing_region.ipynb \
		df-access-pod:/dfdata/extract_contributing_region.ipynb

## Launching the Dask Scheduler and Workers

The [Dask Chart](https://github.com/dask/helm-chart/tree/main/dask) 
for [Helm](https://helm.sh/) is used to deploy a Dask 
scheduler, workers and Jupyter server  (thanks to Ben Weintraub for helping set this up).

The following steps and configuration files were found to work.  First Helm was installed on the local Macbook using homebrew

	brew install helm

and the repo was added:

	helm repo add dask https://helm.dask.org/
	helm repo update

Installing (deploying) requires the yaml file [pysheds_dask_values.yaml](pysheds_dask_values.yaml) 
to limit the requested resources, otherwise the scheduler requests the pods automatically.
The options `jupyter.rbac=False` and `jupyter.serviceAccountName=null` were needed to avoid 
interference with existing installations of Dask running on Nautilus.
The library [pysheds](https://github.com/mdbartos/pysheds) that is installed with this 
configuration is used to calculate the catchment area for debris flow sites, hence the naming.
Other required packages are specified using the environmental option `EXTRA_PIP_PACKAGES`.
Additional changes include setting the password of the jupyter server
(following the [documentation](https://github.com/dask/helm-chart/tree/main/dask)) 
and mounting the debris flows data volume:

	jupyter:
		[...]
        password: 'argon2:enter_your_password_hash_here'
        env:
          - name: EXTRA_PIP_PACKAGES
          value: pandas geopandas pysheds numpy xarray rioxarray matplotlib pyarrow shapely pygeos
        [...]
		mounts:
			volumes:
			- name: data
				persistentVolumeClaim:
					claimName: staley-debrisflow
			volumeMounts:
			- name: data
				mountPath: /home/jovyan/pysheds/

The same is done for `worker.mounts`.  
This [website](https://docs.netapp.com/us-en/netapp-solutions/ai/aks-anf_set_up_dask_with_rapids_deployment_on_aks_using_helm.html) 
was helpful for the configuration of the Helm chart.

The dask chart is installed using

	helm install -f pysheds_dask_values.yaml df-dask dask/dask

and it's status can be verified using:

	helm list

Verify the pods are running:

	kubectl get pods | grep df-dask

The output is something like this:

	df-dask-jupyter-6f8f699557-6r2k4      1/1     Running   0              13m
	df-dask-scheduler-668c6c789c-q5b57    1/1     Running   0              13m
	df-dask-worker-577c8796c4-2tf6l       1/1     Running   0              13m
	df-dask-worker-577c8796c4-pk9b2       1/1     Running   0              13m

Forward port 8888 on the jupyter pod to the local machine:

	kubectl port-forward df-dask-jupyter-6f8f699557-6r2k4 8888:8888

or

	kubectl port-forward deployment/df-dask-jupyter 8888:8888

Next point your browser to the URL http://localhost:8888/lab? to get the Jupyterlab  view.

If you didn't set or don't remember the password, you can obtain the token of the runnning 
server using

    kubectl exec -it deployments/df-dask-jupyter -- jupyter server list

The dashboard runs on the scheduler node on port (8787) and must be forwarded separately:

	kubectl port-forward deployment/df-dask-jupyter 8787:8787

The number of workers is now set to 1 in [helm_dask_values.yaml](helm_dask_values.yaml): 

	worker:
	  replicas: 1

It can be scaled up or down as needed using the command:

	kubectl scale deployment lidar-dask-worker --replicas=10

When done, remove the helm chart using

	helm uninstall df-dask

which removes the jupyter, scheduler and worker deployments.
Please don't forget to scale down the number of workers if the deployement is idle.

**Important**: Save all your results in persistent storage, 
mounted at `/home/jovyan/pysheds/` in this case. 
All other data in the pod (including the user's home directory) will be deleted once
the pod exits or restarts.  As persistent storage is prone to errors, keep regular backups 
from your data in the PVC.

## Configuring the Dask Deployment and the Processing Job

There are a lot of different configuration parameters both in the launching of the dask 
deployment (helm configuration) and in the Dask configuration at runtime.

It's essential to run

	from dask.distributed import Client
	c = Client()

before running computations on Dask bags (or dataframes or array), as the work is 
carried out in serial on the jupyter node otherwise.

Here's a summary of the configuration parameters that can be tweaked for better performance:

## Optional: Tuning the number of threads and workers per pod

The Python processes used in the notebook are typically 
[not threadable](https://distributed.dask.org/en/stable/worker.html).  Therefore, 
only one thread per worker was used (`worker.threads_per_worker = 1`).  As 
pods are always placed on the same physical machine, the number of workers per pod 
can be anywhere from 1 and the maximum number of workers a machine can support. To 
limit the number of pods overall without placing to many workers in one pod,
I used 8 workers per pod.

This behavior can be obtained by setting the resource limit of the worker to 8 and passing the 
extra argument `--nworkers -1` to `dask-worker`, in which case the number of workers is set to 
the number of CPUs.  Note that the number of CPUs is controlled by the CPU resource *limit*, 
not the resource *request*, which may be lower.

        worker:
                [...]
                resources:
                        limits:
                                memory: 64Gi
                                cpu: 8
                        requests:
                                memory: 32Gi
                                cpu: 8
                threads_per_worker: 1
                extraArgs:
                        - --nworkers
                        - "-1"

I initially tested a lower memory request (4Gi) but this resulted in `KilledWorker` errors.  I assume that many similar pods were placed on the same physical machine and exhausting 
the available memory.  The scheduler dashboard showed that some workers used more than 2 Gb of memory.  The cpu request could possibly be set to a value lower than 8, which may result in CPUs being 
shared by different workers.
