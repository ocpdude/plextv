# Plex Media Server w/ MACVLAN
Running a Plex Media Server on OpenShift 4.x

### Build your image
You can build your own image using my Dockerfile as a reference, or just pull the image I created as shaker242/plex:latest. \
`podman build -t ORG/REPO:TAG .`

Notes:
1.  There is a section in the deployment for a 'claim' number... it only lasts 4 mins, I recommend deploying the environment without it to make sure everything is running. Once you have verified the plex pod is running to your liking, obtain your code from https://plex.tv/claim, put it in the deployment and just `oc apply -f plex-deployment.yaml`.
2. You can find your timezone here: https://en.wikipedia.org/wiki/List_of_tz_mediabase_time_zones 
3. You can find the latest version of the Plex Media Server here: https://www.plex.tv/media-server-downloads 

Q. Why MACVLAN? \
A. To improve quality by accessing the Plex Media Server directly, ie put the Pod on the same physical network as the device/s accessing plex. 

Q. Will remote access still work? \
A. Yes. With MACVLAN, I just put in a port-forward from my router to 32400 on the Plex Pod; this needs to be manually adjusted in the "Remote" settings of your Plex Media Server.

Q. What is not covered in this README? \
A. Setting up your networking, I have to leave that mostly to you; however, I have included some notes at the end that may help.

---
## The MACVLAN Setup

> This configuration is used to set a DHCP RANGE of 192.168.11.220-230 for a separate interface (ens224) and not the default (ens192) used for the nodes / vxlan themselves. My setup is OCP 4.5 on VMware vSphere/ESXi 6.7.

1. Create the Network Assignment Definition in the default namespace. \
`oc create -f network-ad.yaml`

  *ens224 is a newly added secondary NIC assigned to my worker nodes.*

2. Verify the new network exists in the default namespace. \
`oc get network-attachment-definitions -n default` 

3. Verify the settings. \
`oc describe network-attachment-definitions macvlan -n default`

4. Create the Plex namespace. \
`oc create ns plex`

5. Use the same RBAC in the main directory. \
`oc -n plex create -f ../plex-rbac.yaml`

6. Apply RBAC to the SA. \
`oc adm policy add-scc-to-user anyuid -z plex-sa -n plex`

7. Test Pod attachment. \
`oc -n plex create -f test-pod.yaml`

8. Verify net1 attached to macvlan. \
`oc -n plex exec test-pod -- ip addr show | grep net1`

```
4: net1@if109: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    inet 192.168.11.220/24 brd 192.168.11.255 scope global net1
```
- ping 192.168.11.220

9. With a successful ping, remove the test pod to prevent an IP conflict. \
`oc -n plex delete -f test-pod.yaml`

>Optional: Create a nfs share for your /media and /config \
`oc -n plex create -f plex-nfs.yaml`

10. Create the Deploymnet. \
`oc -n plex create -f plex-deployment.yaml`

*Since you are using MACVLAN, Services and Routes are not required.*

11. Access your plex server for management: https://192.168.11.220:32400/manage


---
To Do:
1. Detail setting up VMware vSwitch/Port-Groups
2. Detail setting up pfsense 1:1 NAT
3. Detail setting up MACVLAN
4. Consider/Test using a static IP vs DHCP

Notes:
1. In this configuration I have set each of 3 lab nodes to use MACVLAN. If you choose to isolate MACVLAN to a specific node(s), use node lables on your deployment.
2. Once everything checks out, you may have to load a new claim key and reapply the deployment. \
`oc apply -f plex-deployment.yaml`
3. Keeping your /media directory on NFS allows one way to remote mount and upload content.
4. I have removed /transcode since I mostly use plex locallying; thus transcoding isn't used, to improve performance consider /transcode (ing) in memory or NVMe. See plex-deployment.yaml for an example.
5. Each node in my cluster has a second interface (ens224) assigned to it, since DHCP is on this network, I remoted into to each node and reset the secondary interface to non-used IP assignments.
    ```
    Example:
    a. ssh core@worker-0
    b. nmcli connection show
    c. sudo nmcli connection mod 'Wired connection 2' ipv4.method manual ipv4.addresses 10.10.10.10/29 connection.autoconnect yes
    d. ifconfig ens224 10.10.10.10/29 
    e. sudo systemctl restart NetworkManager
    ```

Legacy network to hardset the POD IP, this is not very flexible. \
Install the macvlan addition network. \
`oc edit networks.operator.openshift.io cluster`

```
  additionalNetworks:
  - name: macvlan-plex
    namespace: default
    rawCNIConfig: '{ "cniVersion": "0.3.1", "name": "plex", "master": "ens224", "type":
      "macvlan", "ipam": { "type": "static", "addresses": [ { "address": "192.168.11.200/24",
      "gateway": "192.168.11.1" } ] } }'
    type: Raw
```