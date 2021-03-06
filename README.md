# patroni-class
Master class repo for learning patroni, consul, walg, s3

How to:

# Let's check if we have docker up and running
docker version

# Let's init first 3 containers h0, h1, h2
ansible-playbook -i inventory init.yml

#Let's check if they were deployed ok
docker ps

# Let's check how dynamic_inventory is working
./dynamic_inventory.py

# Now use dynamic_inventory.py as inventory for consul deploy
ansible-playbook -i dynamic_inventory.py consul.yml

# Checking consul cluster is ok
docker logs h0

# Let's perform simple init in postgres cluster
./dynamic_inventory.py # h0->172.18.0.2

# Add one node into cluster with simple init
ansible-playbook -i dynamic_inventory.py patroni.yml --tags=patroni-init --limit=172.18.0.2

# Check if it is up and running
docker exec -it h0 docker ps
docker exec -it h0 docker logs pg-h0

#Lets' check dns records for master
docker exec -it h0 docker exec -it pg-h0 ping master.patroni-class.service.consul

# and deploy slave
ansible-playbook -i dynamic_inventory.py patroni.yml --tags=patroni-init --limit=172.18.0.3

# check if it registred
docker exec -it h0 docker exec -it pg-h0 ping replica.patroni-class.service.consul

# and check logs of first slave
docker exec -it h1 docker logs --tail 100 pg-h1

# Let's check if it data replication is working
# Create table on master
docker exec -it h0 docker exec -it pg-h0 gosu postgres psql -c "CREATE TABLE bins  AS SELECT * FROM GENERATE_SERIES(1, 10000) AS id;"

# Read data from slave
docker exec -it h1 docker exec -it pg-h1 gosu postgres psql -c "SELECT max(id) from bins;"

#Let's perform switchover
docker exec -it h1 docker exec -it pg-h1 bash
patronictl --help

# Check existing state
patronictl -c /var/lib/postgresql/patroni.yml list

# Let's do the trick
patronictl -c /var/lib/postgresql/patroni.yml switchover

# Check existing state
patronictl -c /var/lib/postgresql/patroni.yml list
ping master.patroni-class.service.consul
exit

# Let's disconnect current master
docker network disconnect bridge h1

# Failover works
docker exec -it h0 docker exec -it pg-h0 patronictl -c /var/lib/postgresql/patroni.yml list

# Turn on again and see fencing
docker network connect bridge h1
docker exec -it h0 docker exec -it pg-h0 patronictl -c /var/lib/postgresql/patroni.yml list

# Let's init cluster from s3
# we changed scope from patroni-class to patroni-class-walg
ansible-playbook -i dynamic_inventory.py patroni.yml --tags=patroni-init-walg --limit=172.18.0.4

# and check logs
docker exec -it h2 docker logs --tail 100 pg-h2

# Read data from walg master
docker exec -it h2 docker exec -it pg-h2 gosu postgres psql -c "SELECT max(id) from walg3;"