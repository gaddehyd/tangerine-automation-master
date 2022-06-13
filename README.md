# tangerine-automation 
Workspace for storing all Technical Artifacts for Tangerine's Mortgage Process Automation Engagement. 

## Folder Structure
- CM: Contains all assets related to Case Manager work
- ODM: Contains all assets related to Operational Decision Manager 
- DataCap: Contains all assets from DataCap Development Machine
- DataCap-baskup: Backed up baseline of datacap project without signature capture and mortgage capture
- Mock-Service: Code for mock lookup service used for the project
- Cp4ba-install: Contains assets needed for deploying CP4BA on IBM ROKS

## Problem Context
Tangerine currently uses Quick Modules for ingesting a batch of documents for Mortgage purposes, using their modules to manually ingest the batches, splitting into documents, attaching metadata to the documents and using FileNet Work Baskets/Queues to flow these documents into multiple of these to finally get into fulfillment. Their current process/technology is highly complex and has Quick Modules + FileNet intertwined. Quick Modules does not have any OCR technology underneath. 

## Solution Vision 
![Tangerine-CP4BA-28022022-v2](https://media.github.ibm.com/user/77458/files/93588100-98bb-11ec-8aad-ce936ebe0dde)

### Summary
1. Set of documents get ingested into DataCap via different channels(Email/Fax/Scan/FileServer). Datacap has in-built connectors for all of them to ingest set of documents 
2. Datacap then captures the batch converts it into .tiff, splits the documents & uses image enhancement techniques to prepare the documents. Then it uses the configured ruleSets to classify the documents inside the batch(whether its pay slip, appraisal doc, mortgage pre-approval etc.). Datacap also uses OCR technologies under the hood to extract fields from the documents & attach the metadata to the document(class/subclass). Last step on Datacap end is exporting the classified document along with its metadata to FileNet. 
There are exception processes that might need some human intervention with the documents from Classification/Extraction perspective. 
3. Once the documents are exported to FileNet. FileNet here is used more from a Content Management perspective, think of it as a storage/staging repository for documents(also could be used as archival of documents in ECM capacity). There are appropriate event-triggers configured which trigger corresponding Workflow in BAW
4. Workflow which is part of BAW can now be configured to follow certain steps to accomplish the task at hand. It may include routing the documents to an approver for them to take a look on the interface and approve. It could be a headless workflow/integration to an external system for getting CIF(Client Identifier) or any other information. It could also be a flow with checklist in it to see if all documents have been received by client & send out an email notification to say probably 5/6 documents have been received but one is missing. There are a ton of ways Workflow can help in accomplishing processes. 
5. BAW can also reach out to ODM via a REST call in order to automatically check on a business rule and make a decision on it without any human intervention. 
6. End state for this flow can be probably fulfillment of Mortgage application and documents being archived into FileNet 

## End to end flow diagram
![Tangerine_overall_v3 0](https://media.github.ibm.com/user/77458/files/1efe0700-ae94-11ec-96da-d510da376413)

## Architecture Decisions 

#### Solution: CI Vs CP4BA

Decision was to go with Cloud Pak for Business Automation. Following is the reasoning for the same.
- DataCap fits right into the solution required by the client currently which is more tailored towards classification & extraction
- CI is a GBS asset that can be something which the bank can grow into in near future once the foundation/baseline is set from a classification & extraction perspective
- DataCap(Insights) has a good capability in terms of unstructured documents as well in addition to structured ones
- We also have a CP4BA product, Automated Document Processing, which we will need to compare to the GBS asset
- CI is a cloud-based solution, which the customer is not currently interested in

#### Infrastructure: Client vs IBM Cloud
FileNet is currently present on Client's infra. They also have instances of Case Manager on their infra but not pertaining to this org. Client's current infrastructure is totally on-prem, but as their future roadmap they are looking to move to Google Cloud Platform. It was necessary for them to understand if the solution being proposed will be portable to GCP. There were a lot of processes/access approvals that would have been needed, therefore it was mutually decided that MVP build will be conducted on IBM Infrastructure. We will be using CP4BA installation on ROKS for this MVP. 

#### Service Integration: DataCap Service call vs BAW Integration
Sometimes the CIF is not persent on the document that is being ingested via DataCap, so there needs to be a service all to get CIF(Client identifier) information so it can be used to connect metadata to the document in question. This can either be accomplished via Webservice call using DataCap(actions in DataCap) or it can be taken care of as part of integrations in BAW piece. The ideation & discussion with Architects lead us to a decision of taking this into account on BAW side as this would help keep the responsibilities of systems very clear and provide isolation amongst the components of the solution - DataCap being classification, OCR, extraction mechanism & BAW-ODM being workflow, integrations, Business Rules side of things. 

#### Ingestion channel: Email vs Scan vs Manual Upload vs OLB
Client currently uses Quick Modules for ingestion of the documents via Fax, also needs to scan the paper mail documents manually into the systems. But in case of emails which is major channel for them Quick Modules does not support that channel. So to prove out the value for the high traffic ingestion channel we would like to use Email as the channel for document ingestion for MVP.

#### Number of documents + signature check - ODM vs BAW
As part of the flow there is a need to check if all documents have been received as well as check if signature was properly captured on the document. This could either be done in ODM or can be taken care on BAW side of tooling. As this is not something which would be changed by business in future so doesnt make sense to add this as a business rule. And all the fields, along with signature capture property is available on BAW so it can be easily captured on that end. 

## Technical Lessons Learnt

#### Fixing connectivity issues with DataCap & Filenet

- IBM Content Engine on Demand from ACCE and install it in the Datacap machine - https://www.ibm.com/support/pages/where-and-how-get-filenet-content-engine-net-api
- Download & install Web Services Enhancements (WSE) 3.0 for Microsoft .NET from Official Microsoft Download Center - https://www.microsoft.com/en-us/download/details.aspx?id=14089
- IBM OnDemand Clients V10.5 (download from internal XL download site or fix central)
- Set up SSL connection between a .NET client and FileNet P8 server per instructions - Exporting data: FileNet P8 - https://www.ibm.com/docs/en/datacap/9.1.8?topic=repositories-exporting-data-filenet-p8
- DNS name update


#### Fixing issues with DataCap Plugin for ICN(IBM Content Navigator) 

- For the content route fix, I found a ConfigMap called **cp4a-ban-zen-extension**  with the proxy definition for /icn/navigator.  I also found an identical file named **cp4a-ban-zen-extension.conf** in the /user-home/_global_/nginx-conf.d directory of the ibm-nginx pod.  That directory is mounted from a PVC, making any changes persistent, so I just updated the **cp4a-ban-zen-extension.conf** flie with the following stanza:

```
location /navigator/
{
  access_by_lua_file /nginx_data/checkjwt.lua;
  proxy_set_header Host $host;
  proxy_set_header X-ZEN-ROUTING '$host';
  proxy_pass https://cp4a-navigator-svc.cp4a.svc:9443/navigator/;
  proxy_buffer_size          256k;
  proxy_buffers              8 512k;
  proxy_busy_buffers_size    512k;
  proxy_connect_timeout   300;
  proxy_send_timeout      300;
  proxy_read_timeout      300;
}
```

- The operator copies the contents of the ConfigMap into the nginx deploy during installation, but the operator is turned off right now as it has a tendency to occasionally wreak havoc with the install. For the XSS patch, I updated the **ESAPIWafPolicy.xml** file from the Navigators /opt/ibm/wlp/usr/servers/defaultServer/configDropins/overrides directory with the additional site.  This is in addition to the other changes made to this file to allow scan uploads to work. This directory is also mounted from persistent storage, so the change is permanent.

- The certificates take a bit more work.  I had to extract the base64 value of the **ibm_customBANTrustStore.p12** data element in the cp4a-ban-custom-ssl-secret secret, reconstitute it as a p12 file, add the truststore chain for *.cp4ba.robobob.ca and *.robobob.ca (the cp4ba and datacap AD server identities) to the p12 file, convert it back to base64, and then update the secret with the new value.  This too should be manageable by the operator, but as mentioned before it currently causes more harm than good if left turned on.

Below are the files for first two fixes:

- cp4a-ban-zen-extension.conf

```
location /icn/navigator/
{
  access_by_lua_file /nginx_data/checkjwt.lua;
  proxy_set_header Host $host;
  proxy_set_header X-ZEN-ROUTING '$host';
  proxy_pass https://cp4a-navigator-svc.cp4a.svc:9443/navigator/;
  proxy_buffer_size          256k;
  proxy_buffers              8 512k;
  proxy_busy_buffers_size    512k;
  proxy_connect_timeout   300;
  proxy_send_timeout      300;
  proxy_read_timeout      300;
}
location /navigator/
{
  access_by_lua_file /nginx_data/checkjwt.lua;
  proxy_set_header Host $host;
  proxy_set_header X-ZEN-ROUTING '$host';
  proxy_pass https://cp4a-navigator-svc.cp4a.svc:9443/navigator/;
  proxy_buffer_size          256k;
  proxy_buffers              8 512k;
  proxy_busy_buffers_size    512k;
  proxy_connect_timeout   300;
  proxy_send_timeout      300;
  proxy_read_timeout      300;
}
location /icn/sync/
{
  access_by_lua_file /nginx_data/checkjwt.lua;
  proxy_set_header Host $host;
  proxy_set_header X-ZEN-ROUTING '$host';
  proxy_pass https://cp4a-navigator-svc.cp4a.svc:9443/sync/;
  proxy_buffer_size          256k;
  proxy_buffers              8 512k;
  proxy_busy_buffers_size    512k;
  proxy_connect_timeout   300;
  proxy_send_timeout      300;
  proxy_read_timeout      300;
}
```

- ESAPIWasPolicy.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<policy>
    <aliases />
    <settings>
        <mode>block</mode>
        <error-handling>
            <default-redirect-page>/error.jsp</default-redirect-page>
            <block-status>500</block-status>
        </error-handling>
    </settings>
    <virtual-patches>
        <!-- Uncomment and update to add return host validation when enableOAuthProxy is set to true in web.xml
            <virtual-patch id="oauth2-return" path=".*/jaxrs/oauth2/.*" variable="request.parameters.state" pattern="^(localhost|localhost:http)$" message="Detected invalid OAuth2 return host.">
            </virtual-patch>
            -->
    </virtual-patches>
    <outbound-rules>
        <add-header name="Cache-Control" value="no-cache, no-store" path=".*/|.*\.jsp|.*/jaxrs/.*|.*/api/.*" />
        <add-header name="Content-Security-Policy" value="default-src 'self' blob: https:;
                connect-src 'self' blob: https http://127.0.0.1:* ws://127.0.0.1:* https://cdn.walkme.com:*; 
                font-src 'self' data: blob: https:;
                img-src 'self' data: blob: https:;
                script-src 'self' 'unsafe-inline' 'unsafe-eval' https:;
                worker-src 'self' blob: https:;
                style-src 'self' 'unsafe-inline' https:;
                frame-ancestors 'self'" 
                path="/.*" />
        <!-- <add-header name="Content-Security-Policy"
value="default-src 'self' blob: https:; font-src 'self' data: blob: https:; img-src
'self' data: blob: https:; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; worker-src 'self' blob: https:; style-src 'self' 'unsafe-inline' https:; frame-ancestors 'self'"
path="/.*"/> -->
        <add-header name="Referrer-Policy" value="same-origin" path="/.*" />
        <add-header name="Strict-Transport-Security" value="max-age=7776000; includeSubdomains" path="/.*" />
        <add-header name="X-Content-Type-Options" value="nosniff" path="/.*" />
        <add-header name="X-Frame-Options" value="sameorigin" path="/.*" />
        <add-header name="X-Permitted-Cross-Domain-Policies" value="none" path="/.*" />
        <add-header name="X-XSS-Protection" value="1; mode=block" path="/.*" />
    </outbound-rules>
    <url-rules>
        <!-- Deny HTTP methods other than GET, POST, PUT, DELETE, HEAD, and OPTIONS, and only
allow POST for logon requests -->
        <restrict-method deny="^((?!GET|POST|PUT|DELETE|HEAD|OPTIONS).)*$" />
        <restrict-method deny="^(GET|PUT|DELETE|HEAD|OPTIONS)$" path=".*/jaxrs(/|/.+/)logon$" />
    </url-rules>
</policy>
```

#### Business Objects in Case Manager creates issues in activity flows(unable to save the object values)

Team wanted to use Customer Object to save all the properties into a business object and use it across our activities for the Case. We saw issues on the logging end when we tried to use the object & the values were available on one activity but not the other one - causing discrepencies & distress amongst the team.The options to fix that or work around is as below:
- Create data objects on bpmn end and map the properties into this object and use it across the activities
- Flatten out all properties into single ones(non-object) and use them across activities on the Case

#### General Tips from specialist Rob Peeren on BAW
- Create trap error blocks, as a bpm good practice
- Put logic as much in your services as possible, not in high level process exception: coach because high level process blocks writes to db and kill performance
- Do things in services is easier to debug. Individual testing cannot be done in the process.
- Objects should have default value for easier unit testing.

#### Datacap general tips
- If you are running Datacap on VMs please try to keep isolation between different teams 
- Take regular backups of VMs along with disk snapshots so you can rollback to stable state if something goes wrong

## Reference Links
Tangerine Enagement Box folder - https://ibm.ent.box.com/folder/149858877113?s=dkr83s4e8te20bdlfct0kebz4uj4qg6s

CI(Content Intelligence Asset) - https://github.ibm.com/cognitive-insurance-platform/getting-started

DataCap + FileNet Activation - https://ibm.ent.box.com/notes/910991891644

DataCap RedBook - http://www.redbooks.ibm.com/redbooks/pdfs/sg247969.pdf

DataCap Documentation - https://www.ibm.com/docs/en/datacap/9.1.7

BAW Lab - https://ibm.ent.box.com/v/CP4AutoDaL20-1MatForPart/file/717883037227

Accelerators Link - https://ibm.ent.box.com/v/DBATechnicalSalesAccelerators/folder/43184191746

BAW Workflow Lab - https://github.com/IBM/cp4ba-labs/blob/main/Workflow/Lab%20Guide%20-%20Introduction%20to%20IBM%20Business%20Automation%20Workflow.pdf
