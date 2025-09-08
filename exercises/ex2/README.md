# Exercise 2 - Exactly Once In Order Scenario using partitioned Queues

In this exercise, we will model and run an integration flow supporting Exactly Once In Order delivery by using an **partitioned Queue on SAP Integration Suite, Advanced Event Mesh (AEM)**. This ensures the following:
- By persisting the messages in an AEM queue, Cloud Integration can carry out the retry of the message delivery in case of an error.
- By using a queue, the message sequence can be preserved
- By using a partitioned queue, we enable parallelization to a certain extend (depends on the number of partitions)

Situation with **exclusive JMS Queue**: A message fails and is retried automatically until the message is successfully delivered. As long as the failed message is in retry, **all successor messages** are kept on hold to ensure that they don't overtake the predecessor message.
situation with **partitioned AEM Queue**: A message fails and is retried automatically until the message is successfully delivered. As long as the failed message is in retry, **all successor messages in the same partition** (same Partition Key Hash) are kept on hold to ensure that they don't overtake the predecessor message.

In the exercise, you won't start from scratch, instead you will use a template so that you can focus on the Exactly Once In Order specific settings only. In our example integration flow, a message is sent to an **SAP RM sender adapter** and directly stored in a partitioned queue on AEM to guarantee message retry in case of a message processing error. The SAP RM protocol extends the plain SOAP protocol to support Exactly Once and Exactly Once In Order delivery by providing SAP proprietary SOAP headers or query parameters. To support Exactly Once, a **message id** needs to be passed to the integration flow. For Exactly Once In Order delivery, you need to transfer a **queue id** to the integration flow. If a queue ID is present, the quality of service is implicitly determined as Exactly Once In Order. 

There difference to exercise 1 is that AEM is not running inside of the Integration Suite Tenant, so you´ll need to provide **connection details**. Additionally to the Queue ID on AEM you´ll also need to provide the User Property **"JMSXGroupID"** to AEM as Key Value for the generation of the Partition Key Hash. Based on this the messages are distributed across the queue partitions.

Since AEM offers a lot of settings for Queues, In contrast to exercise 1 the Queue is not just created by maintaining it in the channel. You actively have to open up the **AEM Broker Manager** and create the Queue explicitly + predefine how many partitions and consumers you need. For EOIO the most important parameter here is **"Maximum Delivered Unacknowledged Messages per Flow"** in Advanced Settings. If the value here is not 1, EOIO is not guaranteed. 

The second integration process reads the message from the very same AEM queue and runs the actual integration logic, in our case a message mapping, but only if the QueueID = "12345". The Routing condition is added here to demonstrate, that if the mapping is not deployed, the Messages in the affected Queue Partition are stuck, until the mapping is deployed and the first message is processed. Until this is the case, the messages from other partitions can still be processed, because they contain different Partition Key Hashs (based on the QueueID).

Once the message mapping has been successfully carried out, the message is reliably exchanged with a receiver using the XI 3.0 protocol. The **XI adapter** in Cloud Integration doesn’t natively support Exactly Once In Order delivery. Instead, you need to select the **Handled by Integration Flow** delivery assurance to implement the same. If you use the Handled by Integration Flow delivery assurance setting, the XI receiver adapter doesn’t persist the outgoing message which is not needed here because the message is persisted and retried from the exclusive JMS queue anyway. Furthermore, the XI receiver adapter expects the headers **SapQualityOfService** and **SapQueueId** to be set within the integration flow. Those headers are actually configured by using the corresponding headers passed from the SAP RM sender adapter.

For an improved monitoring of the exchanged messages, we have configured the corresponding SAP headers and custom header properties. This is already part of the provided template.

In order to simulate the error situation, we will use a re-usable message mapping in the integration flow as global resource which we intentionally won't deploy in the first place. To resolve the error, we will simply deploy the message mapping artifact. The failed message and all successor messages which were on hold will be eventually successfully delivered after an automatic retry.

## Exercise 1.1 - Undeploy Message Mapping from Exercise 1

Since we are using the same Message Mapping, this needs to be undeployed before starting with Exercise 2.

Go to Monitoring -> Integrations and APIs-> Manage Integration Content. Then filter based on your MessageMapping Name MM EOIO - XX

<img width="2543" height="584" alt="image" src="https://github.com/user-attachments/assets/6a73c2ff-ecc9-4550-aff5-68a8e8f23d23" />

## Exercise 1.2 - Copy Provided Templates

The copying process was explained in Exercise 1. Please copy the Iflow "EOIO Partitioned Queue - Template" analogous to the first Iflow into your Package.

## Exercise 1.3 - Model your Integration Flow

Now that you have copied the provided templates, we should be all set to enhance the copied integration flow model to ensure Exactly Once In Order delivery.
    
1.  In your package, select the copied integration flow **EOIO Partitioned Queue - XX** with **XX** the ID assigned to you to open the model designer.

<img width="1064" height="469" alt="image" src="https://github.com/user-attachments/assets/158cefc2-4869-4821-a478-a154fa02a6ea" />
    
2. The integration flow contains two integration processes: the **Integration Process: Provider flow** to store the incoming message in the AEM queue, and the **Integration Process: Consumer flow** to read the message from the very same AEM queue. In the integration flow designer, switch to **Edit** mode.

<img width="2270" height="769" alt="image" src="https://github.com/user-attachments/assets/c7e8ae4d-9446-44e7-b71b-399996abfbab" />

3. First, we will maintain the AEM connection to decouple the two integration processes. In the editor, select the message end event **End** of the upper integration process **Integration Process: Provider flow**, then drag ...

<img width="1072" height="293" alt="image" src="https://github.com/user-attachments/assets/864c2b25-ca6e-4141-bc20-b744956eebd4" />
  
4. ... and drop a connection to the Receiver.

<img width="1257" height="434" alt="image" src="https://github.com/user-attachments/assets/07c347a7-ba1e-4081-88ba-567578d5f0f5" />

5. In the upcoming dialog, select the Adapter Type **AdvancedEventMesh**. 

<img width="1025" height="366" alt="image" src="https://github.com/user-attachments/assets/a25e7b6c-5c9e-49a3-9633-856fee55aaed" />
   
6. Select the AEM connection. In the Properties section of the AEM receiver adapter, switch to the **Connection** tab, and click on "externalize":

Please Name your Externalized Parameters for all of the Parameters

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | {{AEM_Host}} | 
| MessageVPN           | {{AEM_MessageVPN}}                                          |
| Username              | {{AEM_Username}}                                         |
| AuthenticationType   | {{AEM_AuthenticationType}}                                                         |
| PasswordSecureAlias | {{AEM_PasswordSecureAlias}}                                               |

<img width="996" height="457" alt="image" src="https://github.com/user-attachments/assets/b0d784b3-4d18-4fc7-801d-85f9d9326357" />
   
7. Select the **Processing** Tab. Change the Parameters:

| Name              | Value                       |
| ----------------- | --------------------------- |
| Delivery Mode     | Persistent (Guaranteed)     |
| Endpoint Type     | Queue                       |
| Destination Name  | HOW_EOIO_PQ_{{participant}} |

Note that we are here externalizing parts of the parameters so we can only assign one value and use them in other parts as well (e.g. in the Receiver and Sender Channel of the AEM Connection)

```yaml
{{participant}}
```

8. In the **Message Properties** Tab please add these Properties:

|Name                   | Value                       |
| ----------------------| ----------------------------|
| Sender ID         | ${header.SAP_Receiver}    |

   In the User Properties please add the JMSXGroupID and purchaseOrder

|Key            | Value                         |    
| --------------| ----------------------------|
| JMSXGroupID    | ${header.SapPlainSoapQueueId} |
| purchaseOrder    | ${header.purchaseOrder} |
   

9. Scroll down to the **Integration Process: Consumer flow**, and add a AEM sender connection between the Sender and the message start event. On the tab **Connection** of the JMS sender adapter, maintain the same externalized parameters as in the AEM Receiver Channel in the upper Flow.

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | {{AEM_Host}} | 
| MessageVPN           | {{AEM_MessageVPN}}                                          |
| Username              | {{AEM_Username}}                                         |
| AuthenticationType   | {{AEM_AuthenticationType}}                                                         |
| PasswordSecureAlias | {{AEM_PasswordSecureAlias}}                                               |
  
10. In the **Processing** Tab please maintain following parameters:

- "Run on a single worker node" = active
- "Parallel consumers" = 10
- "Queue Name = "HOW_EOIO_PQ_{{participant}}"
- "Acknowledgement Mode" = "Automatic On Exchange Complete" 

<img width="1107" height="508" alt="image" src="https://github.com/user-attachments/assets/1d1ef04f-f1e0-4ea8-a423-ee80ecaba6d9" />

11. Next, we need to add the message mapping. Click outside of the integration processes to be able to configure the integration flow components. From the integration flow configuration, switch to tab **References**. Within the references, switch to tab **Global**, and select Message Mapping from the **Add References** menu.

<img width="2271" height="825" alt="image" src="https://github.com/user-attachments/assets/8eb5037f-71b4-4914-92a7-97b7a2b02f28" />

12. In the upcoming **References** dialog, select the beforehand added message mapping **MM EOIO - XX** and select **OK**.

<img width="1568" height="994" alt="image" src="https://github.com/user-attachments/assets/f15b4176-97a8-474d-96b0-6126fec59516" />

13. The message mapping should be assigned behind the Router condition in the Step "Message Mapping" in the **Integration Process: Consumer flow**. Select the flow step **Message Mapping**, select the **Processing** Tab and click Select.

<img width="1695" height="877" alt="image" src="https://github.com/user-attachments/assets/a6db52a9-008d-4a8b-9df9-e9348b1d48cf" />

14. Choose the "Global Ressources" Tab and Select the Message Mapping "MM_EIOI_<your participant id>". In my case MM_EOIO_XX:

<img width="922" height="271" alt="image" src="https://github.com/user-attachments/assets/2e0bb9ea-dfb1-415e-9438-3cb0f0d2f85e" />

15. XI Receiver: This was already set up in Ex. 1. In this Iflow everything is prepared on XI Receiver already, so nothing to do there.

16. Save your Integration Flow as a Version.

## Exercise 1.4 - Configure Queue on AEM Broker

1. Open up the Broker Manager here:
  
   https://mr-connection-h91kb3o1b6w.messaging.solace.cloud:943/?_gl=1*v2iwkf*_gcl_au*MTQ1MTI1NTAyNC4xNzU2OTAxMDU0LjU1OTA2MDg5Ni4xNzU3MzMyOTk3LjE3NTczMzMwMTc.*_ga*MTc3MjY3NjI1OS4xNzExNjMzNDUy*_ga_XZ3NWMM83E*czE3NTczNDEzMzMkbzI2NiRnMSR0MTc1NzM0MTM0OSRqNDQkbDAkaDA.#/msg-vpns/YWVtX2NvbW11bml0eWNlbnRyYWw=?token=YWJj.eyJhY2Nlc3NfdG9rZW4iOiAibWlzc2lvbi1jb250cm9sLW1hbmFnZXI6NjJlYWZzZTNmdTdrOHEyZjFkdmdvMXM1cjEifQ%3D%3D.eHl6&title=AEM_CommunityCentral&subtitle=aem_communitycentral

2. Create a Queue with Name "HOW_EOIO_PQ_<your participant Number", e.g. HOW_EOIO_PQ_XX
   <img width="2542" height="385" alt="image" src="https://github.com/user-attachments/assets/193fce22-8659-466d-97c4-5ffde2ac4831" />

3. Change the Access Type to "Non-Exclusive and the Partition Count as well as the Maximum Consumer Count to 10
   <img width="1210" height="670" alt="image" src="https://github.com/user-attachments/assets/72802c59-0f44-4863-bcaf-8d33c63a6e17" />

4. Open up Advanced Settings

<img width="2541" height="643" alt="image" src="https://github.com/user-attachments/assets/b77e466c-dbdd-4c11-98b3-554146e13783" />
   
5. Change "Maximum Delivered Unacknowledged Messages per Flow" to 1 (needed for EOIO)
<img width="1570" height="955" alt="image" src="https://github.com/user-attachments/assets/7c50ba43-d9a9-4303-9aa8-9d7172cc02b0" />

6. Apply your configuration

7. Just FYI: The required connection details for the CLoud Integration Flow can be derived from the AEM Cluster Manager (in the AEM Dashboard outside of the Broker Manager)

   <img width="2555" height="905" alt="image" src="https://github.com/user-attachments/assets/46d50e3f-e6e0-42e4-888f-7cc42d4459fb" />

The connection details are provided in the Table in the next Step.

## Exercise 1.3 - Configure and Deploy your Integration Flow

In the following, you will configure and deploy the beforehand modified integration flow.
    
1.  After having saved and canceled, you should see the **Configure** button on the upper right. Select **Configure**. Here you can set up the Connection Details for AEM and your participent Number:

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| AEM_Host                  | tcps://mr-connection-h91kb3o1b6w.messaging.solace.cloud:55443 | 
| AEM_MessageVPN           | aem_communitycentral                                          |
| AEM_Username              | solace-cloud-client                                           |
| AEM_AuthenticationType   | Basic                                                         |
| AEM_PasswordSecureAlias | RampUp_AEM_User                                               |

<img width="1152" height="494" alt="image" src="https://github.com/user-attachments/assets/564f0720-1689-4827-ab8c-e83008798609" />

The "Password Secure Alias" is the a Basic User Credential defined in Monitor -> Manage Security -> Security Material

<img width="1582" height="918" alt="image" src="https://github.com/user-attachments/assets/4d1e6550-ccc2-41d5-ad27-2313b53278db" />

    
2. In the configuration dialog, on tab **Sender**, same as for Exercise 1: maintain the value of the **participant** parameter. Replace **XX** of the preconfigured value **UserXX** with the number assigned to you. The value of the parameter is actually appended to the integration flow end point to ensure a unique end point deployed on the tenant. Furthermore, the value is part of the partitioned Queue Name on AEM. Then **Save** and **Deploy**.

<br>![image](/exercises/ex1/images/01_03_Deploy_02.png)

3. Once deployed, you should see a toast message on the bottom of the screen. Furthermore, the **Runtime Status** on top should show as **Started**.

<br>![image](/exercises/ex1/images/01_03_Deploy_03.png)

4. You can also check the deployment status in the monitoring. Navigate to **Monitor > Integrations and APIs** from the menu, and select the tile **Manage Integration Content**.

<br>![image](/exercises/ex1/images/01_03_Deploy_04.png)

5. In the **Manage Integration Content** page, filter for your ID **XX**. You should see your deployed integration flow in status **Started**.

<br>![image](/exercises/ex1/images/01_03_Deploy_05.png)

Now you are all set to test your scenario!

## Exercise 1.5 - Test and Monitor

To test your configuration scenario, we use the Bruno API client application for which we have provided a collection with pre-configured sample requests. As a prerequisite to test your integration scenario using the Bruno API client, you should have gone through [Setup Bruno API client](../ex0#setup-bruno-api-client/). If not, do the setup, then come back and proceed with the steps below.

1. Open the Bruno application on your laptop, expand the **EOIO Hands-on Workshop** collection and the **EOIO via Exclusive Queue** folder. Select the **Post Order in Sequence to AEM** POST request. Ensure that the right environment is selected which defines the host name of the tenant, the client id and the secret. Depending on which tenant you use in the exercises, select either **eu-02a** or **eu-02b**. In the provided URL, the **participant** variable should hold your user number <b>XX</b> assigned to you assuming that you have properly configured the parameter in the Bruno API client setup. The message GUID passed to the integration flow is automatically generated. Trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code **202 Accepted**. In our first Test we leave the QueueID = "12345", so the Mapping does not get executed.

<img width="2559" height="974" alt="image" src="https://github.com/user-attachments/assets/179c053c-741c-4639-a55e-e8aea6c3c848" />

2. Navigate back to the monitoring page of Cloud Integration, and select the link **Monitor Message Processing**

<img width="1346" height="735" alt="image" src="https://github.com/user-attachments/assets/658da656-c994-4ff5-92c9-2a79eb6ed553" />

Then Filter for your Iflow

<img width="1962" height="308" alt="image" src="https://github.com/user-attachments/assets/da7ea13d-3dcd-44d1-ba9e-5c87a75b962c" />

You can also filter based on QueueID

<img width="2533" height="890" alt="image" src="https://github.com/user-attachments/assets/5b380d09-63df-4888-b104-5541f31c6d0a" />

or purchaseOrder (Custom Header = OrderID)

<img width="2500" height="937" alt="image" src="https://github.com/user-attachments/assets/a41fee96-744c-4f58-81e8-ff8ba50864a8" />

You should see 2 messages here: the first (bottom) is the message from the client to AEM, the second (top) is the Consumer from AEM to XI. You can identify it based on the Sender and Receiver Header.

<img width="1970" height="666" alt="image" src="https://github.com/user-attachments/assets/0eae798f-687c-4fe9-884c-4b0be15e620e" />

Our Mapping is only triggered when QueueID != 12345. So let´s send another 2 Messages from Bruno with QueueID = 123456 and 2 more with QueueID = 67891 

<img width="2138" height="514" alt="image" src="https://github.com/user-attachments/assets/1a18236f-fd16-480f-9d38-c3715fd7a4a7" />

You should see now fore each QueueID two Message with Status "Completed". These are the Messages From the Client to AEM. Then there should be 1 Retry Message. This means AEM will try to push the message again in certain periods (defined in the Queue in AEM Broker Management).

Let´s check in AIM (https://mr-connection-h91kb3o1b6w.messaging.solace.cloud:943/?_gl=1*qknv9f*_gcl_au*MTQ1MTI1NTAyNC4xNzU2OTAxMDU0LjU1OTA2MDg5Ni4xNzU3MzMyOTk3LjE3NTczMzMwMTc.*_ga*MTc3MjY3NjI1OS4xNzExNjMzNDUy*_ga_XZ3NWMM83E*czE3NTczNDEzMzMkbzI2NiRnMSR0MTc1NzM0MjcwMCRqMzkkbDAkaDA.#/msg-vpns/YWVtX2NvbW11bml0eWNlbnRyYWw=?token=YWJj.eyJhY2Nlc3NfdG9rZW4iOiAibWlzc2lvbi1jb250cm9sLW1hbmFnZXI6NjJlYWZzZTNmdTdrOHEyZjFkdmdvMXM1cjEifQ%3D%3D.eHl6&title=AEM_CommunityCentral&subtitle=aem_communitycentral) 

<img width="2028" height="648" alt="image" src="https://github.com/user-attachments/assets/0786d61d-4053-4400-8113-2b302aa09eae" />

We can see that AEM has distributed the Messages to Partition 2 and 6. You can also see more if you  click one partition and on "Messages Queued". 

<img width="2545" height="565" alt="image" src="https://github.com/user-attachments/assets/d82d05e4-d2f4-4dd1-8001-f25d914c025a" />

This means that these 2 Partitions are blocked until the error is resolved. If I send another Message with QueueID = "12345", it should go through, unless it is assigned to one of the blocked partitions. Let´s try it:

<img width="2138" height="501" alt="image" src="https://github.com/user-attachments/assets/92536e05-8e09-411c-9fb3-34825c266f04" />

We can see in the Message Monitoring that it is completed:
<img width="2172" height="429" alt="image" src="https://github.com/user-attachments/assets/0128c097-ac29-4497-b1eb-9da447a32c21" />

Now let´s deploy the Message Mapping again:

<img width="2307" height="492" alt="image" src="https://github.com/user-attachments/assets/4d73e5ea-f18c-4632-bd50-fac07bd94e3e" />

Now after Refresh on AEM the Queues are clear:

<img width="2325" height="609" alt="image" src="https://github.com/user-attachments/assets/525de94c-8416-4f6f-bf40-ace342249f9e" />

and Messages with QueueID 123456 & 67891 are completed

<img width="2257" height="870" alt="image" src="https://github.com/user-attachments/assets/38a150b2-e351-42fe-a1a3-3c612b427fa5" />



## Summary

Congratulations. You have successfully modelled and tested an Exactly Once In Order scenario using Partitioned Queues.
