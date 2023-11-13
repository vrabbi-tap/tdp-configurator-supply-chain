# Tanzu Developer Portal (TDP) Configurator Supply Chain
This repo contains the needed files to apply to your cluster to use the TDP configurator custom supply chain.  
While you can use the default supply chain provided in TAP, this custom supply chain gives a better UX, and automates additional elements of the currently manual process.
  
This supply chain includes on the needed steps, and does not include the additional steps from the standard supply chains which are not relevant to our use case.

## Setting up the supply chain
To create the supply chain, you must do the following steps:
1. Update the values.yaml file in the root of this repo with your specific values
2. Render the setup manifests using YTT  
```bash
ytt -f values.yaml -f setup_env.yaml > tdp-configurator-environment.yaml
```  
3. Apply the environment setup to your cluster  
```bash
kubectl apply -f tdp-configurator-environment.yaml
```  
4. Render the supply chain based on those values using YTT  
```bash
ytt -f supplychain/ -f values.yaml > rendered-supply-chain.yaml
```  
5. Apply the manifests to the cluster
```bash
kubectl apply -f templates/
kubectl apply -f rendered-supply-chain.yaml
```  
6. Now you can create the supply chain like before with 2 small differences from the documented workload:
* you must set the label **apps.tanzu.vmware.com/workload-type: "tdp"** on the workload as that is this supply chains selector
* you can omit the **source.image** field and the **BP_NODE_RUN_SCRIPTS** and **TPB_CONFIG** env vars as they will be defaulted by the supply chain.

## Example Workload YAML
```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  labels:
    app.kubernetes.io/part-of: tdp-configurator
    apps.tanzu.vmware.com/workload-type: tdp
  name: tdp-configurator
spec:
  build:
    env:
    - name: TPB_CONFIG_STRING
      value: YXBwOgogIHBsdWdpbnM6CiAgICAtIG5hbWU6ICdAdm13YXJlLXRhbnp1L3RkcC1wbHVnaW4tdGVjaGluc2lnaHRzJwogICAgICB2ZXJzaW9uOiAnMC4wLjInCiAgICAtIG5hbWU6ICdAdm13YXJlLXRhbnp1L3RkcC1wbHVnaW4taG9tZScKICAgICAgdmVyc2lvbjogJzAuMC4yJwogICAgLSBuYW1lOiAnQHZtd2FyZS10YW56dS90ZHAtcGx1Z2luLXN0YWNrLW92ZXJmbG93JwogICAgICB2ZXJzaW9uOiAnMC4wLjInCiAgICAtIG5hbWU6ICdAdm13YXJlLXRhbnp1L3RkcC1wbHVnaW4tZ2l0aHViLWFjdGlvbnMnCiAgICAgIHZlcnNpb246ICcwLjAuMicKICAgIC0gbmFtZTogJ0B2bXdhcmUtdGFuenUvdGRwLXBsdWdpbi1wcm9tZXRoZXVzJwogICAgICB2ZXJzaW9uOiAnMC4wLjInCiAgICAtIG5hbWU6ICdAdm13YXJlLXRhbnp1L3RkcC1wbHVnaW4tYmFja3N0YWdlLWdyYWZhbmEnCiAgICAgIHZlcnNpb246ICcwLjAuMicKYmFja2VuZDoKICBwbHVnaW5zOgogICAgLSBuYW1lOiAnQHZtd2FyZS10YW56dS90ZHAtcGx1Z2luLXRlY2hpbnNpZ2h0cy1iYWNrZW5kJwogICAgICB2ZXJzaW9uOiAnMC4wLjInCiAgICAtIG5hbWU6ICdAdm13YXJlLXRhbnp1L3RkcC1wbHVnaW4tbGRhcC1iYWNrZW5kJwogICAgICB2ZXJzaW9uOiAnMS4wLjAnCg==
```

Alternatively, you can use the `tpb_plugins` parameter to specify the plugin configuration in plain text. This allows you to directly edit your plugin configuration in a GitOps repo, without the need for base64-encoding.
```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  labels:
    app.kubernetes.io/part-of: tdp-configurator
    apps.tanzu.vmware.com/workload-type: tdp
  name: tdp-configurator
spec:
  params:
    - name: tpb_plugins
      value: |
        app:
          plugins:
            - name: '@vmware-tanzu/tdp-plugin-techinsights'
              version: '0.0.2'
            - name: '@vmware-tanzu/tdp-plugin-home'
              version: '0.0.2'
        backend:
          plugins:
            - name: '@vmware-tanzu/tdp-plugin-techinsights-backend'
              version: '0.0.2'
            - name: '@vmware-tanzu/tdp-plugin-ldap-backend'
              version: '1.0.0'
```


7. Copy the secret to the tap-install namespace  
* While this can be done in many different ways, here we will show an example using the carvel secret gen controller which is already being used in TAP.
## Exporting the secret
We will first export as an environment variable the namespace where we have deployed the tdp configurator workload:
```bash
export TDP_CONFIGURATOR_NAMESPACE=<FILL ME IN>
```  

Next we will create a secret export that allows us o import the created secret into the tap-install namespace where it must reside in order to be used as an overlay:
```bash 
cat <<EOF | kubectl apply -f -
---
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretExport
metadata:
  name: tdp-app-image-overlay-secret
  namespace: $TDP_CONFIGURATOR_NAMESPACE
spec:
  toNamespace: tap-install
EOF
```  

Next we will create a secret import resource in the tap-install namespace in order to clone the secret.
```bash
cat <<EOF | kubectl apply -f -
---
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretImport
metadata:
  name: tdp-app-image-overlay-secret
  namespace: tap-install
spec:
  fromNamespace: $TDP_CONFIGURATOR_NAMESPACE
EOF
```  

Now any time we rebuild a new TDP image, the secret will automatically be synced to the tap-install namespace and our TDP deployment will rollout a new revision with our new image!

The final step is to do as mentioned in the docs, and add the package overlay stanza in your TAP values, as well as any additional backstage configurations your plugins may require!
