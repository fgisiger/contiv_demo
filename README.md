##------------------- before demo, check current config on ACI 


# ------------------- specify aci mode and vlan range
netctl global set --fabric-mode aci --vlan-range 850-900
netctl global info


# ------------------- Create a tenant named MS1
netctl tenant create MS1
netctl tenant ls


# ------------------- Choose the subnet and bd that you like...

netctl net create -t MS1 -e vlan -s 10.0.0.0/24 -g 10.0.0.1 bd1
netctl net ls -t MS1

# ------------------- Creating two EPGs : app and web
netctl group create -t MS1 bd1 app
netctl group create -t MS1 bd1 web

netctl group ls -a


# ------------------- Push app-profile to ACI 

netctl app-profile create -t MS1 -g app,web ap1
netctl app-profile ls -t MS1

##------------------- check config on APIC.  ap profile is pushed with two EPGs, pysycial domains, static binding and etc



# ------------------- Creating containers with app/MS1 and web/MS1 as a network

docker network ls    (network name is epg/tenant name )

docker run -itd --net="app/MS1" --name=app1 jainvipin/web /bin/bash
docker run -itd --net="app/MS1" --name=app2 jainvipin/web /bin/bash
docker run -itd --net="web/MS1" --name=web1 jainvipin/web /bin/bash

# ------------------- Running docker ps command to check docker container creation

docker ps -a


##------------------- now confirm that, app1 container can talk to app2, but NOT to web1 as no contracts in place


# ------------------- Testing the policies and rules which we have applied on app and web groups.

# ------------------- Now create ICMP allow policy so that these container can ping each other.

netctl policy create -t MS1 web2app
netctl policy rule-add -t MS1 -d in --protocol icmp  --from-group web  --action allow web2app 1
netctl group create -t MS1 -p web2app bd1 app


# ------------------- now confirm that app1/app2 container CAN ping web1 container


# ------------------- Create external contracts and associate it with app EPG

netctl external-contracts create --tenant MS1 -c --contract "uni/tn-MS1/brc-app2db" appConsumed
netctl group create -t MS1 -e appConsumed -p web2app bd1 app

##------------------- useful netctl commands

netctl version
etcdctl cluster-health
netctl global inspect
netctl network inspect -t MS1 bd1
netctl group inspect -t MS1 app
netctl group inspect -t MS1 web

docker logs -f <aci-gw container_id>

service aci-gw status
service netmaster status


# ------------------- cleaning the testbed

docker rm -f $(docker ps -a | grep web | awk '{print $1}')
netctl app-profile rm -t MS1 ap1
netctl group rm -t MS1 app
netctl group rm -t MS1 web
netctl external-contracts rm --tenant MS1  appConsumed
netctl policy rm -t MS1 web2app
netctl network rm -t MS1 bd1
netctl tenant rm MS1


netctl policy rule-add -t MS1 -d in --protocol tcp --port 6379 --from-group app  --action allow web2app 2
