== GPU Accelerated SQL queries with PostgreSQL & PG-Strom in OpenShift-3.10 
Fonte: https://blog.openshift.com/gpu-accelerated-sql-queries-with-postgresql-pg-strom-in-openshift-3-10/




# prepare images
git clone https://github.com/giofontana/openshift-psap.git
yum -y install buildah skopeo podman

# prepare postgresql
yum -y install libhugetlbfs-utils
hugeadm --create-global-mounts
cp openshift-psap/blog/gpu/pg-strom/pgstrom.conf /etc/tuned/pgstrom/tuned.conf
tuned-adm profile pgstrom
reboot

oc new-project nvidia
oc create -f openshift-psap/playbooks/roles/nvidia-device-plugin/files/nvidia-device-plugin-scc.yaml
oc label node <your-gpu-node> openshift.com/gpu-accelerator=true

mkdir -p /opt/psql-data
chmod 777 /opt/psql-data

oc create -f openshift-psap/blog/gpu/pg-strom/pgstrom.yml
oc rsh pgstrom /bin/bash
vi /var/lib/pgsql/data/userdata/postgresql.conf # add section below

#------------------------- snip -----------------------------------------
## postgresql.conf
huge_pages = on
# Initial buffers and mem too small, increase it to work in mem
# and not in storage
shared_buffers = 30GB
work_mem = 30GB
# PG-Strom internally uses several background workers,
# Default of 8 is too small, increase it
max_worker_processes = 100
max_parallel_workers = 100
# PG-Strom module must be loaded on startup
shared_preload_libraries = '/usr/pgsql-10/lib/pg_strom.so,pg_prewarm'
#------------------------- snip -----------------------------------------

oc replace --force -f openshift-psap/blog/gpu/pg-strom/pgstrom.yml