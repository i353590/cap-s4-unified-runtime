#S/4 Business Logic Extn Scenario – Migration to Unified Runtime  

   

Status: Migration is complete now and the app works as expected. 

   

Steps to Migrate:  

   

Clone https://github.com/SAP-samples/cloud-extension-s4hana-business-process  

Follow steps : https://github.com/SAP-samples/cloud-extension-s4hana-business-process/blob/mission/mission/configure-oData-Service/README.md to configre Odata service and technical user creation  

Setup connectivity : Setup connectivity between S/4HANA system, SAP BTP  

   

Open terminal and run “mbt build -t ./”  

In the root folder create a folder name as “kyma-s4-html5-app-deployer”  

Under the folder “kyma-s4-html5-app-deployer” create “package.json” file and add the below code  

{  

    "name": "kyma-sf-html5-app-deployer",  

    "version": "1.0.0",  

    "dependencies": {  

      "@sap/html5-app-deployer": "4.1.2"  

    },  

    "scripts": {  

      "start": "node node_modules/@sap/html5-app-deployer/index.js"  

    }  

}  

   

Copy the “resources” folder into the folder “kyma-s4-html5-app-deployer.”  

Make changes in the mta.yaml like below https://github.tools.sap/I307469/sf-unified-runtime/blob/main/mta.yaml  

IMPORTANT: Convert all the instances and application’s name to LOWERCASE as Kyma doesn’t support UPPERCASE characters.  

Connectivity service plan – lite not available, we need to use connectivity_proxy plan  

Logging service is not available in kyma, it has to be removed from mta.  

   

Use bumblebee to transform the mta to helm like below “bee transform mta --registry <your_registry> --source mta.yaml --out helm –force”  

Example: bee transform mta --registry s700136 --source mta.yaml --out helm –force  

Note: s700136 is the registry name please use your own registry to create helm  

   

Open values-dev.yaml under helm folder and add the below code  

kyma-sf-html5-app-deployer:  

    env:  

        SAP_CLOUD_SERVICE : "com.sap.bp.BusinessPartners.one"  

        SUBACCOUNT_LEVEL_DESTINATION: "true"  

        BACKEND_DESTINATIONS:  

            secretKeyRef:  

                name: backend-destinations  

                key: BACKEND_DESTINATIONS  

Add the parameters for Enterprise messaging service like below  

epm:  

    parameters:  

        emname: bpems  

        namespace: refapps/bpems/abc  

        options:  

            management: true  

            messagingrest: true  

            messaging: true  

        rules:  

            queueRules:  

                publishFilter:  

                - ${namespace}/*  

                subscribeFilter:  

                - ${namespace}/*  

            topicRules:  

                publishFilter:  

                - ${namespace}/*  

                subscribeFilter:  

                - ${namespace}/*  

        version: 1.1.0  

   

   

   

login to your kyma and create secret for binding to your service layer  

 Note: create namespace for “s4”  

 Example : https://github.tools.sap/I307469/sf-unified-runtime/blob/main/secrets/backend-destinations.json  

   

run the below command to create secret : kubectl create  secret generic backend-destinations --from-file=BACKEND_DESTINATIONS=./secrets/backend-destinations.json -n <namespace>  

Example: kubectl create  secret generic backend-destinations --from-file=BACKEND_DESTINATIONS=./secrets/backend-destinations.json -n cl  

note: cl is my namespace  

   

Application logging service is not available in Kyma – Have to be removed.  

Connectivity service plan ‘lite’ is not available in Kyma – Needs to be changed to “connectivity_proxy”. Also, Connectivity Service should be only one per cluster, so that needs to be taken care as well. Create a connectivity instnace, then create a binding manually. 

I have used the name as businesspartnervalidation-cs for both the instance and binding. Moments after the binding is created, a secret will also be created with same name of that binding. 

Create a file to create a configMap to use connectivity proxy. 

		Example:  

 

apiVersion: v1 

kind: ConfigMap 

metadata: 

name: s4-new-connectivity-proxy-info 

namespace: cl 

data: 

.metadata: >- 

{"credentialProperties":[{"name":"subaccount_id","format":"text"},{"name":"subaccount_subdomain","format":"text"},{"name":"token_service_domain","format":"text"},{"name":"token_service_url","format":"text"},{"name":"token_service_url_pattern","format":"text"},{"name":"token_service_url_pattern_tenant_key","format":"text"},{"name":"clientid","format":"text"},{"name":"credential-type","format":"text"},{"name":"xsappname","format":"text"},{"name":"clientsecret","format":"text"},{"name":"connectivity_service","format":"json"},{"name":"onpremise_proxy_host","format":"text"},{"name":"onpremise_proxy_http_port","format":"text"},{"name":"onpremise_proxy_host","format":"text"},{"name":"url","format":"text"}],"metaDataProperties":[{"name":"instance_name","format":"text"},{"name":"instance_guid","format":"text"},{"name":"plan","format":"text"},{"name":"label","format":"text"},{"name":"type","format":"text"},{"name":"tags","format":"json"}]} 

onpremise_proxy_host: connectivity-proxy.kyma-system.svc.cluster.local 

onpremise_proxy_http_port: "20003" 

onpremise_proxy_ldap_port: "20001" 

onpremise_proxy_port: "20003" 

onpremise_proxy_rfc_port: "20001" 

onpremise_socks5_proxy_port: "20004" 

 

Note: The ‘name’ field in the file is the configmap name which we will pass in the values.yaml file. 

 

 Update the values.yaml file like this under srv application: 

 

additionalVolumes: 

- name: connectivity-secret 

volumeMount: 

mountPath: /bindings/connectivity 

readOnly: true 

projected: 

sources: 

- secret: 

name: businesspartnervalidation-cs 

optional: false 

- secret: 

name: businesspartnervalidation-cs 

optional: false 

items: 

- key: token_service_url 

path: url 

- configMap: 

name: s4-new-connectivity-proxy-info 

optional: false 

 

Note: ‘connectivity-secret’ is my secret name and ‘s4-new-connectivity-proxy-info' is the configmap name we created earlier. 

HANA instance mapping to Kyma: MAP HANA instance to Kyma so that we can create db instances in Kyma cluster. 

now run “helm dependency update ./helm” to update the dependencies. 

deploy the helm by running the below command “helm install s4-unified ./helm --values helm/values-dev.yaml --namespace cl”  

note: if above command doesn’t work then make changes like below file to deploy : https://github.tools.sap/I307469/sf-unified-runtime/blob/main/helm/values.yaml  

   

Configure event communication between S/4 and Event Mesh: https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/810dfd34f2cc4f39aa8d946b5204fd9c/fbb2a5980cb54110a96d381e136e0dd8.html?version=1809.000  

Configure Cloud connector: https://github.com/SAP-samples/cloud-extension-s4hana-business-process/blob/mission/mission/cloud-connector/README.md  

   

to run the demo please follow this : https://github.com/SAP-samples/cloud-extension-s4hana-business-process/blob/mission/mission/testbasicscenario/README.md  

   

   

   

Issues:  

   

https://github.tools.sap/unified-runtime/bumblebee/blob/ea1349f13985e4eb76a21a50bd27aa2273b10cd5/pkg/helm/manifest.go#L296 – Mb to Mi Bee doesn’t do.  

Need to make sure that the mta has the unit as ‘M’ like this - https://github.tools.sap/unified-runtime/bumblebee/blob/f8f17a9dbe404197fd27e9ac4cf74b4d4ff3f707/examples/simple-mta/mta.yaml#L48  

   

Type in mta is not supported : type: com.sap.application.content  

It should be replaced with :  type: com.sap.html5.application.content  

But for kyma need more changes in mta where cf deployment is different from kyma deployment like below after creating html5-deployer  

  - name: kyma-s4-html5-app-deployer  

type: com.sap.html5.application.content  

path: kyma-s4-html5-app-deployer  

requires:  

- name: businesspartnervalidation-html5-repo-host  

- name: businesspartnervalidation-xsuaa  

- name: businesspartnervalidation-dest  

parameters:  

content-target: true  

build-parameters:  

timeout: 20m  

requires:  

- name: comsapbpbusinesspartners  

artifacts:  

- './*'  

target-path: resources/comsapbpbusinesspartners  

  

   

destination types are not supported so need to identify the deployed service url and create secret.  

Note: URL property must be changed as per the kyma cluster  

https://github.tools.sap/I307469/sf-unified-runtime/blob/main/secrets/backend-destinations.json  

 
