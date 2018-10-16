# Istio Sample for gRPC Transcoding

In this sample, I am using the ProductCatalog gRPC microservice from [here](https://github.com/GoogleCloudPlatform/microservices-demo), use Istio to transcode, so we could access this via http/json

### Pre-requisites
Existing Kubernetes cluster and Istio installed

The instructions here are for GCP

#### Step 1 
[Document](https://cloud.google.com/endpoints/docs/grpc/transcoding) here explain how you can annotate your proto file and generate transcoding friendly APIs

Once done, Follow instructions [here](https://cloud.google.com/endpoints/docs/grpc/transcoding#ensure_http_rules_are_deployed) to generate your *.pb descriptor file

Refer to `demo.http.proto` for my http annotations

Envoy needs the proto descriptor to transcode, so we need to find a way to supply this file to Envoy (istio-proxy).

#### Step 2

(I am using PersistentDisk, because ConfigMap couldn't handle the binary descriptor file)

Create a GCE Persistent disk and copy your proto descriptor file (*.pb) to this disk

For eg, you can create a disk using command like this
```
gcloud compute disks create --size=1GB --zone=us-central1-a istio-disk

```
Once done, copy the descriptor file to this disk. I attached it a GCE instance and scp'ed the file to this disk

#### Step 3

We will now update the istio-sidecar-injector config map and add a volume mount to this Persistent disk
```
      volumes:
      - name: gce-disk
        gcePersistentDisk:
          pdName: istio-disk
          readOnly: true
          fsType: ext4
```
```
    volumeMounts:
    - mountPath: /gce
      name: gce-disk  
```
Update the ConfigMap
```
kubectl apply -f istio-sidecar-injector.yaml
```

#### Step 4

Apply the Envoy filter

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: hipster-product-grpc
  namespace: istio-system
spec:
  workloadLabels:
    transcode: http
  filters:
  - listenerMatch:            
      listenerType: SIDECAR_INBOUND
    filterName: envoy.grpc_json_transcoder
    filterType: HTTP
    filterConfig:
      proto_descriptor: "/gce/api_descriptor.pb"
      services:         
        - hipstershop.ProductCatalogService      

```
#### Step 5

Now deploy the service
```
kubectl apply -f products.yaml
```

Now you can access the Product catalog service with http/json

```
curl productcatalogservice.default.svc.cluster.local/products
curl productcatalogservice.default.svc.cluster.local/products/OLJCESPC7Z
```

