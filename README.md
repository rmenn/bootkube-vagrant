# bootkube-vagrant
Coreos kubernetes cluster with bootkube

#### Instructions
- Grab the bootkube 0.13.0 verstion [here][1]
- Render `./bootkube render --asset-dir cluster --api-servers=https://controller-1.coreos.local:6443 --etcd-servers=https://controller-1.coreos.local:2379 --network-provider=experimental-canal`
- ensure the rendered folder is in the same directory as the vagrant file
- Zip the directory `zip -r cluster.zip cluster/`
- Add the master ip to `/etc/hosts` run `sudo echo "172.17.4.100 controller-1.coreos.local" > /etc/hosts`  
- `vagrant up`

#### Caveats

- etcd runs as single node on controller-1
