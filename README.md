I - setup LAB
You must setup a lab OCP 4.2+ to deploy Service Mesh. You can use lab from opentlc:
- OpenShift 4 Service Mesh Lab

In our case we will use a standard OpenShift platform to start from the installation procedure.

You can use service mesh foundations lab guide from learning platform.
In this procedure we will only make use of lab guide scripts, but will go through another path.
$ git clone https://github.com/thoraxe/lab-ossm.git

II - Deploy Service Mesh Operator
1 - Install operators
We will make use of the official procedure:
https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_install/installing-ossm.html

Steps:
- Installing the Elasticsearch Operator
- Installing the Jaeger Operator
- Installing the Kiali Operator
- Installing the Red Hat OpenShift Service Mesh Operator

2 - Deploy Service Mesh control plane
$ oc new-project my-smcp

In the following control plane configuration is a basic configuration
$ oc create -f control_plane.yaml

In order to apply a custom configuration for each component we recommend to follow the guides in official documentation:
https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_install/customizing-installation-ossm.html#ossm-cr-example_customizing-installation-ossm

The must important element to retain, is to configure jeager in a production elasticsearch rather than keeping all-in-one mode for dev, or demo purprose.

3 - Create ServiceMeshMemberRoll
Once control plane is deployed and running, administrator needs to configure wich namespaces need to become part of the cluster.
A CRD can be created to list all the namespace members of the cluster.

Please notice that users who dont have admin role can create ServiceMeshMember  to add projects into the Service Mesh cluster.
administrator can then give the privileges to do so.
https://docs.openshift.com/container-platform/4.5/service_mesh/service_mesh_install/installing-ossm.html#ossm-member-roll-create_installing-ossm

$ oc create -f member_roll.yaml

3 - Deep dive in security
Community istio make use of envoy as side proxy to apply routing rules. Envoy proxy uses cap_net_admin privileges with init sidecars to manipulate iptables of nodes when application runs.
Red Hat Service Mesh consider this configuration as a security vulnerability, thus it makes use of istio-cni as a deamonset named istio-node.
This deamon set is only owned by cluster-admin, and it's this component that allows to manipulate the iptables of the underlaying nodes.

$ oc get daemonset istio-node -n openshift-operators

4 - Deploy the sample lab
$ oc new-project my-tutorial
$ oc create -n my-tutorial -f curl.yaml 
$ oc create -n my-tutorial -f customer.yaml
$ oc create -n my-tutorial -f preference.yaml
$ oc create -n my-tutorial -f recommendation.yaml

5 - Envoy injector
Once the lab deployed, you can remark that all service mesh pods had a envoy side car proxy injected automatically.
$ oc get pod -l app=customer -o yaml| grep -i 'name: istio-envoy'

This is thank to the kubernetes webhooks mutation mechanism, which is implemented by the istio operator in our use case.
$ oc get mutatingwebhookconfiguration istio-sidecar-injector-my-smcp

III - Routing, Rules and Policies
1 - Gateway, VirtualService & Destination Rules
Istio control plane exposes an ingress gateway to the cluster, in order to extend the configuration to route the traffic into our applications,
We need first to create a gateway, and a VirtualService.
$ oc create -f default_routing_demo.yaml

The gateway will make the application accessible from outside the cluster on a specific host.
$ oc get gateway customer-gateway -o yaml

The VirtualService will applies a routing rules and redirects to the specific backend.
$ oc get virtualservice customer -o yaml

Request customer service
$ export MY_TUTORIAL_GATEWAY=$(oc get route my-tutorial -n my-smcp -o 'jsonpath={.spec.host}')
$ curl $MY_TUTORIAL_GATEWAY/customer

2 - A/B testing (with Weighted Routing configuration)
Check Gateway, Virtual Service you can see that traffic is splitted 80% to V1, 20% V2, 0% V3
---
  - route:
    - destination:
        host: recommendation
        subset: v1
      weight: 80
    - destination:
        host: recommendation
        subset: v2
      weight: 20
---

Check that all requests are distributed accoding to configuration, and take a look on Kiali dashbord to observe
$ while true; do curl $MY_TUTORIAL_GATEWAY/recommendation; sleep 1; done
recommendation v1 from '5fcfd9bb6c-kg6qb': 180
recommendation v2 from '7fd6798df9-zfphz': 120

In this case we are calling recommendation directly to check the behaviour, ideally we could separate the rules in two different 
virtual services. But for demo purpose we configured it that way. In next exemple, we'll create an additional virtual service
to validate the call through customer service

3 - Canary Release
$ oc create -f recommendation_canary.yaml

Check that all requests are distributed accoding to configuration, and take a look on Kiali dashbord to observe
$ while true; do curl -H "user-location: Boston" $MY_TUTORIAL_GATEWAY/customer; sleep 1; done
customer => preference => recommendation v2 from '7fd6798df9-zfphz': 163
customer => preference => recommendation v2 from '7fd6798df9-zfphz': 164

$ while true; do curl $MY_TUTORIAL_GATEWAY/customer; sleep 1; done
customer => preference => recommendation v1 from '5fcfd9bb6c-kg6qb': 203
customer => preference => recommendation v1 from '5fcfd9bb6c-kg6qb': 204

4 - Mirroring traffic
$ oc apply -f recommendation_mirroring.yaml

$ oc get pods -l app=recommendation,version=v3 --no-headers | awk '{print $1}'
recommendation-v3-666b94f647-2b75h

$ oc logs -f recommendation-v3-666b94f647-2b75h -c recommendation
...
Oct 02, 2020 12:59:08 PM com.redhat.developer.demos.recommendation.rest.RecommendationResource getRecommendations
INFO: recommendation request from 666b94f647-2b75h: 105

$ while true; do curl $MY_TUTORIAL_GATEWAY/customer; sleep 1; done
customer => preference => recommendation v2 from '7fd6798df9-zfphz': 175
customer => preference => recommendation v2 from '7fd6798df9-zfphz': 176

$ oc logs -f recommendation-v3-666b94f647-2b75h -c recommendation
...
Oct 02, 2020 1:02:11 PM com.redhat.developer.demos.recommendation.rest.RecommendationResource getRecommendations
INFO: recommendation request from 666b94f647-2b75h: 107

IV - Advanced configuration
1 - Retries, Timeouts and Circuit Breaking

2 - Rate Limiting/Policy

3 - Fault Injection

V - Security
1 - mTLS

2 - Authentication

3 - OPA simple
