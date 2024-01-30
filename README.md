# Plex Media Server
[![PlexTV Builder](https://github.com/ocpdude/plextv/actions/workflows/plextv-builder.yaml/badge.svg)](https://github.com/ocpdude/plextv/actions/workflows/plextv-builder.yaml) [![Linting Dockerfiles 🐳](https://github.com/ocpdude/plextv/actions/workflows/dockerfile-lint.yaml/badge.svg)](https://github.com/ocpdude/plextv/actions/workflows/dockerfile-lint.yaml) [![Linting Kubernetes ☸️](https://github.com/ocpdude/plextv/actions/workflows/kubernetes-lint.yaml/badge.svg)](https://github.com/ocpdude/plextv/actions/workflows/kubernetes-lint.yaml)

Running a Plex Media Server on OpenShift 4.x 
1. The Default Setup - this build covers deploying plex as a native container with a pod, service and route.
2. The MACVLAN Setup - this build covers deploying plex as a native containter tied to a physical network interface and static ip address. \
*Visit /macvlan for this setup*

### Build your image
You can build your own image using my Dockerfile as a reference, or just pull the image I created as shaker242/plex:latest. \
`podman build -t ORG/REPO:TAG .` 

Notes:
1.  There is a section in the deployment for a 'claim' number... it only lasts 4 mins, I recommend deploying the environment without it to make sure everything is running. Once you have verified the plex pod is running to your liking, obtain your code from https://plex.tv/claim, put it in the deployment and just `oc apply -f plex-deployment.yaml`.
2. You can find your timezone here: https://en.wikipedia.org/wiki/List_of_tz_mediabase_time_zones 
3. You can find the latest version of the Plex Media Server here: https://www.plex.tv/media-server-downloads 


## The Default Setup

1. Create your namespace/project \
`oc new-project plex --display-name "Plex Media Server"`

2. Apply RBAC and create your SA \
`oc create -f ./manifests/plex-rbac.yaml`

3. Apply scc-anyuid to your SA in the plex namespace \
`oc adm policy add-scc-to-user anyuid -z plex-sa -n plex`

4. Create the PVC's for /config /media /transcode \
`oc create -f ./manifests/plex-nfs.yaml`

5. Deploy Plex \
`oc create -f ./manifests/plex-deployment.yaml`

5. Link the services to our Deployment \
`oc create -f ./manifests/plex-services.yaml`

6. Once everything checks out, you will have to load your claim key and reapply the deployment. \
`oc apply -f ./manifests/plex-deployment.yaml`

7. Plex is up, volumes are connected but now you'll need to install your routes and configure your media server. \
`oc create -f ./manifests/plex-routes.yaml`

8. Remote access, for this out of scope item you will have to create a port forward from your wan/ip to this container... mine goes like this \
WAN/IP -> firewall port forward (port WAN/22400 -> port LAN LB/32400 NodePort) -> LB 32400 to Node:32400 -> Pod/Container:32400 \
The external port 22400 is then manually set in Plex's remote settings.

9. Example: NFS Exports
```
/plex/config    *(rw,sync,no_subtree_check,no_wdelay,no_root_squash,insecure)
/plex/transcode	*(rw,sync,no_subtree_check,no_wdelay,no_root_squash,insecure)
/plex/media     *(rw,sync,no_subtree_check,no_wdelay,all_squash,anonuid=65534,anongid=65534,insecure)	
```
*Note: Remote mount the nfs volume (nfsserver:/plex/media) to upload, download and delete content.

10. Example: Directory Permissions \
`chown -R /plex` \
`chown nobody:nogroup /plex/media` 

## If you'd like to improve this, please submit a PR!