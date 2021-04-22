# Running Gloo Mesh on OpenShift
_Running Gloo Mesh on OpenShift._



## Installing Gloo Mesh
Assign the license key.   
```
GLOO_MESH_LICENSE_KEY="$(cat ~/Playground/licensing/mesh.key)" 
```

Add the gloo-mesh-enterprise helm chart and install gloo mesh with prometheus enabled.  
```
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise  

helm repo update  

k create ns gloo-mesh  

helm install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise --namespace gloo-mesh --set enterprise-networking.metricsBackend.prometheus.enabled=true --set licenseKey=$GLOO_MESH_LICENSE_KEY  
```

## Preparing OpenShift for Istio/Gloo Mesh
`oc adm policy add-scc-to-group anyuid system:serviceaccounts:gloo-mesh`  
`oc adm policy add-scc-to-user anyuid -z gloo-mesh`  
`oc adm policy add-scc-to-group privileged system:serviceaccounts:gloo-mesh`  
`oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-system`  
`oc adm policy add-scc-to-group privileged system:serviceaccounts:default`  
`oc adm policy add-scc-to-group anyuid system:serviceaccounts:default`  
```
cat <<EOF | oc -n default create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: istio-cni
EOF
```  

<!-- ```
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: scc-admin
allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
- cw-admin
groups:
- my-admin-group
``` -->

## Install Istio in each cluster

Management Cluster
```
cat << EOF | istioctl manifest install -y --context $MGMT_CONTEXT -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: openshift-istiooperator
  namespace: istio-system
spec:
  profile: openshift
  meshConfig:
    enableAutoMtls: true
    defaultConfig:
      envoyMetricsService:
        address: enterprise-agent.gloo-mesh:9977
      envoyAccessLogService:
        address: enterprise-agent.gloo-mesh:9977
      proxyMetadata:
        # Enable Istio agent to handle DNS requests for known hosts
        # Unknown hosts will automatically be resolved using upstream dns servers in resolv.conf
        ISTIO_META_DNS_CAPTURE: "true"
        GLOO_MESH_CLUSTER_NAME: mgmt-cluster
  components:
    # Istio Gateway feature
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        env:
          - name: ISTIO_META_ROUTER_MODE
            value: "sni-dnat"
        service:
          type: NodePort
          ports:
            - port: 80
              targetPort: 8080
              name: http2
            - port: 443
              targetPort: 8443
              name: https
            - port: 15443
              targetPort: 15443
              name: tls
              nodePort: 32001
  values:
    global:
      multiCluster:
        clusterName: mgmt-cluster
      pilotCertProvider: istiod
EOF
```

## Update Enterprise Networking
Path the enterprise networking to be of type loadbalancer
```
kubectl -n gloo-mesh patch svc enterprise-networking -p '{"spec": {"type": "LoadBalancer"}}' 
```  

## Patch the istio ingressgateway to be of type loadbalancer   
```
kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec": {"type": "LoadBalancer"}}' --context $MGMT_CONTEXT
kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec": {"type": "LoadBalancer"}}' --context $REMOTE_CONTEXT
```

## Register Clusters
```
SVC=$(kubectl --context $MGMT_CONTEXT -n gloo-mesh get svc enterprise-networking -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):9900

meshctl cluster register enterprise   --remote-context=$REMOTE_CONTEXT   --relay-server-address $SVC  remote-cluster 
meshctl cluster register enterprise   --remote-context=$MGMT_CONTEXT   --relay-server-address $SVC  mgmt-cluster
```


## Scale Down
```
k scale deploy/dashboard --replicas=0 -n gloo-mesh 
k scale deploy/enterprise-networking --replicas=0 -n gloo-mesh 
k scale deploy/prometheus-server --replicas=0 -n gloo-mesh 
k scale deploy/rbac-webhook --replicas=0 -n gloo-mesh 
```



## DeRegister Cluster
```
✗ meshctl cluster deregister enterprise remote-cluster
✗ meshctl cluster deregister enterprise mgmt-cluster
```


## Uninstall Helm Chart 
```
helm uninstall -n gloo-mesh gloo-mesh-enterprise
```


Deploy Bookinfo Application in Management Cluster:  
```
kubectl --context $MGMT_CONTEXT label namespace default istio-injection=enabled
# deploy bookinfo application components for all versions less than v3
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version notin (v3)'
# deploy all bookinfo service accounts
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account'
# configure ingress gateway to access bookinfo
kubectl --context $MGMT_CONTEXT apply -f https://raw.githubusercontent.com/istio/istio/1.8.2/samples/bookinfo/networking/bookinfo-gateway.yaml
````


## Upgrade 
Install the CRDS
```
https://raw.githubusercontent.com/solo-io/gloo-mesh/v1.1.0-beta1/install/helm/gloo-mesh-crds/crds/discovery.mesh.gloo.solo.io_v1_crds.yaml
```

Uninstall Gloo
```
helm uninstall -n gloo-mesh gloo-mesh-enterprise
```
Upgrade the helm chart
```
helm upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise  --version 1.1.0-beta3 --namespace gloo-mesh --set enterprise-networking.metricsBackend.prometheus.enabled=true --set licenseKey=$GLOO_MESH_LICENSE_KEY   --force
```