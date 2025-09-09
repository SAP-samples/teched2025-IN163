# Exercise 2 - Exactly Once In Order Scenario using partitioned Queues

In this exercise, we will model and run an integration flow supporting Exactly Once In Order delivery by using an **partitioned Queue on SAP Integration Suite, Advanced Event Mesh (AEM)**. This ensures the following:
- By persisting the messages in an AEM queue, Cloud Integration can carry out the retry of the message delivery in case of an error.
- By using a queue, the message sequence can be preserved.
- By using a partitioned queue, we enable parallelization to a certain extend (depends on the number of partitions).

The situation with **exclusive JMS Queue** is as follows: A message fails and is retried automatically until the message is successfully delivered. As long as the failed message is in retry, **all successor messages** are kept on hold to ensure that they don't overtake the predecessor message.

When using **partitioned AEM Queues**, the behavior in case of an erroneous message slightly differs from using **exclusive JMS Queues**: As long as the failed message is in retry, **all successor messages in the same partition** (that is, with the same Partition Key Hash) are kept on hold to ensure that they don't overtake the predecessor message.

In the exercise, you won't start from scratch, instead you will use a template so that you can focus on the Exactly Once In Order specific settings only. In our example integration flow, a message is sent to an **SAP RM sender adapter** and directly stored in a partitioned queue on AEM to guarantee message retry in case of a message processing error. The SAP RM protocol extends the plain SOAP protocol to support Exactly Once and Exactly Once In Order delivery by providing SAP proprietary SOAP headers or query parameters. To support Exactly Once, a **message id** needs to be passed to the integration flow. For Exactly Once In Order delivery, you need to transfer a **queue id** to the integration flow. If a queue ID is present, the quality of service is implicitly determined as Exactly Once In Order. 

The difference to Exercise 1 is that AEM is not running inside of the SAP Integration Suite tenant, so you need to provide **connection details**. Additionally to the Queue ID on AEM, you also need to provide the User Property **JMSXGroupID** to AEM as Key Value for the generation of the Partition Key Hash. Based on this, the messages are distributed across the queue partitions.

Because AEM offers a lot of configuration options for setting up the Queues, in contrast to exercise 1 the Queue is not just created by maintaining it in the channel. You actively have to open up the **AEM Broker Manager** and create the Queue explicitly. Furthermore, you need to predefine how many partitions and consumers are required. For ensuring EOIO, the most important parameter here is **Maximum Delivered Unacknowledged Messages per Flow** in the Advanced Settings. If the value of this parameter is not 1, EOIO can not be guaranteed. 

The second integration process reads the message from the very same AEM queue and runs the actual integration logic, in our case a message mapping, but only if the QueueID **not** equals **12345**. The routing condition is added here to simulate an error situation: if the message mapping is not deployed, the messages in the corresponding Queue Partition stuck until the mapping is deployed and the first message is processed. If you choose the QueueID **12345**, first of all the messages won't run the message mapping and hence won't run into an error in the first place. Secondly, the messages are most likely distributed to a different partition because of a different generated Partition Key Hash (based on the QueueID) and hence are not blocked.

Once the message mapping has been successfully carried out, the message is reliably exchanged with a receiver using the XI 3.0 protocol. The **XI adapter** in Cloud Integration doesn’t natively support Exactly Once In Order delivery. Instead, you need to select the **Handled by Integration Flow** delivery assurance to implement the same. If you use the Handled by Integration Flow delivery assurance setting, the XI receiver adapter doesn’t persist the outgoing message which is not needed here because the message is persisted and retried from the exclusive JMS queue anyway. Furthermore, the XI receiver adapter expects the headers **SapQualityOfService** and **SapQueueId** to be set within the integration flow. Those headers are actually configured by using the corresponding headers passed from the SAP RM sender adapter.

For an improved monitoring of the exchanged messages, we have configured the corresponding SAP headers and custom header properties. This is already part of the provided template.

In order to simulate the error situation, we will use a re-usable message mapping in the integration flow as global resource which we intentionally won't deploy in the first place. To resolve the error, we will simply deploy the message mapping artifact. The failed message and all successor messages which were on hold will be eventually successfully delivered after an automatic retry.

## Exercise 2.1 - Undeploy Message Mapping from Exercise 1

Since we use the same Message Mapping like in Exercise 1, this needs to be undeployed before starting with Exercise 2 to be able to force an error situation.

Navigate to **Monitoring > Integrations and APIs > Manage Integration Content**. Then filter based on your message mapping Name **MM EOIO - XX**. Select **Undeploy** to undeploy your message mapping artifact.

<img width="2543" height="584" alt="image" src="https://github.com/user-attachments/assets/6a73c2ff-ecc9-4550-aff5-68a8e8f23d23" />

## Exercise 2.2 - Copy Provided Template

In the following, you will copy the provided integration flow to your package. As you did this before in Exercise 1, we won't describe this in detail here. Run through the similar steps like in Exercise 1.1 to copy the integration flow template **EOIO Partitioned Queue - Template** to your package **User XX** with **XX** the ID assigned to you. As name for the copied integration flow, choose **EOIO Partitioned Queue - XX**. This time, you do not need to copy the message mapping template as we will reuse the message mapping from Exercise 1.

## Exercise 2.3 - Model your Integration Flow

Now that you have copied the provided template, we should be all set to enhance the copied integration flow model to ensure Exactly Once In Order delivery.
    
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

6. Select the AEM connection. In the **Properties** section of the AEM receiver adapter, click on **Externalize**. On the **Externalization** dialog, stay on tab **Connection**, and maintain the Externalized Parameters as follows. When done, select **OK** to close the **Externalization** dialog.

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | {{AEM_Host}} | 
| MessageVPN           | {{AEM_MessageVPN}}                                          |
| Username              | {{AEM_Username}}                                         |
| AuthenticationType   | {{AEM_AuthenticationType}}                                                         |
| PasswordSecureAlias | {{AEM_PasswordSecureAlias}}                                               |

<img width="996" height="457" alt="image" src="https://github.com/user-attachments/assets/b0d784b3-4d18-4fc7-801d-85f9d9326357" />
   
7. Switch to the **Processing** tab. Maintain the Parameters as follows:

| Name              | Value                       |
| ----------------- | --------------------------- |
| Delivery Mode     | Persistent (Guaranteed)     |
| Endpoint Type     | Queue                       |
| Destination Name  | HOW_EOIO_PQ_{{participant}} |

Note that we are here externalizing parts of the parameters so we can only assign one value and use them in other parts as well (e.g. in the Receiver and Sender Channel of the AEM Connection)

```yaml
{{participant}}
```

8. Switch to the **Message Properties** tab and add the following Properties:

|Name                   | Value                       |
| ----------------------| ----------------------------|
| Sender ID         | ${header.SAP_Receiver}    |

Scroll down to the **User Properties** section, add the **JMSXGroupID** and **purchaseOrder** properties.

|Key            | Value                         |    
| --------------| ----------------------------|
| JMSXGroupID    | ${header.SapPlainSoapQueueId} |
| purchaseOrder    | ${header.purchaseOrder} |
   

9. In the integration flow model, scroll down to the **Integration Process: Consumer flow**, and add an AEM sender connection between the Sender and the message start event. In the **Properties** section of the AEM sender adapter, click on **Externalize** and maintain the same externalized parameters as in the AEM Receiver Channel before. When done, select **OK** to close the **Externalization** dialog.

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | {{AEM_Host}} | 
| MessageVPN           | {{AEM_MessageVPN}}                                          |
| Username              | {{AEM_Username}}                                         |
| AuthenticationType   | {{AEM_AuthenticationType}}                                                         |
| PasswordSecureAlias | {{AEM_PasswordSecureAlias}}                                               |
  
10. Switch to the **Processing** tab, and maintain the following parameters:

| Name              | Value                       |
| ----------------- | --------------------------- |
| Run on a single worker node     | Select the check box    |
| Parallel consumers     | 10                       |
| Queue Name  | HOW_EOIO_PQ_{{participant}} |
| Acknowledgement Mode  | Automatic On Exchange Complete |

<img width="1107" height="508" alt="image" src="https://github.com/user-attachments/assets/1d1ef04f-f1e0-4ea8-a423-ee80ecaba6d9" />

11. Next, we need to add the message mapping. Click outside of the integration processes to be able to configure the integration flow components. From the integration flow configuration, switch to tab **References**. Within the references, switch to tab **Global**, and select Message Mapping from the **Add References** menu.

<img width="2271" height="825" alt="image" src="https://github.com/user-attachments/assets/8eb5037f-71b4-4914-92a7-97b7a2b02f28" />

12. In the upcoming **References** dialog, select the beforehand added message mapping **MM EOIO - XX** and select **OK**.

<img width="1568" height="994" alt="image" src="https://github.com/user-attachments/assets/f15b4176-97a8-474d-96b0-6126fec59516" />

13. The message mapping should be assigned behind the Router condition in the Step **Message Mapping** in the **Integration Process: Consumer flow**. Select the flow step **Message Mapping**, select the **Processing** tab and click **Select**.

<img width="1695" height="877" alt="image" src="https://github.com/user-attachments/assets/a6db52a9-008d-4a8b-9df9-e9348b1d48cf" />

14. Choose the **Global Ressources** tab and select the beforehand added Message Mapping **MM_EOIO_XX** with **XX** the user ID assigned to you. Select **OK**.

<img width="922" height="271" alt="image" src="https://github.com/user-attachments/assets/2e0bb9ea-dfb1-415e-9438-3cb0f0d2f85e" />

15. **Save** your Integration Flow.

## Exercise 2.4 - Configure Queue on AEM Broker

1. Open up the Broker Manager here:
  
   https://mr-connection-h91kb3o1b6w.messaging.solace.cloud:943/?_gl=1*v2iwkf*_gcl_au*MTQ1MTI1NTAyNC4xNzU2OTAxMDU0LjU1OTA2MDg5Ni4xNzU3MzMyOTk3LjE3NTczMzMwMTc.*_ga*MTc3MjY3NjI1OS4xNzExNjMzNDUy*_ga_XZ3NWMM83E*czE3NTczNDEzMzMkbzI2NiRnMSR0MTc1NzM0MTM0OSRqNDQkbDAkaDA.#/msg-vpns/YWVtX2NvbW11bml0eWNlbnRyYWw=?token=YWJj.eyJhY2Nlc3NfdG9rZW4iOiAibWlzc2lvbi1jb250cm9sLW1hbmFnZXI6NjJlYWZzZTNmdTdrOHEyZjFkdmdvMXM1cjEifQ%3D%3D.eHl6&title=AEM_CommunityCentral&subtitle=aem_communitycentral

2. Create a Queue with Name **HOW_EOIO_PQ_UserXX** with **XX** the user ID assigned to you.

3. Change the Access Type to **Non-Exclusive** and the **Partition Count** as well as the **Maximum Consumer Count** to **10**.
<img width="1168" height="611" alt="image" src="https://github.com/user-attachments/assets/98761960-8401-4821-baff-e3eb5d2d8514" />

4. Open up the **Advanced Settings**.

<img width="1553" height="629" alt="image" src="https://github.com/user-attachments/assets/fd1dc9d7-f9ad-4bb4-9528-884dabed7431" />
   
5. Change the parameter **Maximum Delivered Unacknowledged Messages per Flow** to **1** (needed for ensuring EOIO).

<img width="1532" height="505" alt="image" src="https://github.com/user-attachments/assets/e28f3ba5-7a7d-480c-ac5b-b3a00d791819" />

6. **Apply** your configuration.

7. FYI: The required connection details for the Cloud Integration Flow can be derived from the AEM Cluster Manager (in the AEM Dashboard outside of the Broker Manager).

   <img width="2555" height="905" alt="image" src="https://github.com/user-attachments/assets/46d50e3f-e6e0-42e4-888f-7cc42d4459fb" />

Note: The connection details are provided in the table below.

## Exercise 2.5 - Configure and Deploy your Integration Flow

In the following, you will configure and deploy the beforehand modified integration flow.
    
1.  Navigate back to your integration flow, and select the **Configure** button on the upper right. If you can't see the button, you may first have to **Cancel** assuming that you have first saved your changes. Select **Configure**. In the upcoming dialog, stay on tab **Sender** and select the Sender **AEM_Sender** from the drop down menu. Set up the Connection Details for AEM and maintain your participant number as follows:

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | tcps://mr-connection-h91kb3o1b6w.messaging.solace.cloud:55443 | 
| Message VPN           | aem_communitycentral                                          |
| AEM_Username          | solace-cloud-client                                           |
| Authentication Type   | Basic                                                         |
| Password Secure Alias | RampUp_AEM_User                                               |
| participant           | UserXX with XX your user id                                   |

Note: Because we use externalized parameters, the connection details for the AEM_Receiver adapter is automatically set.

2. When done, **Save** and **Deploy**.

<img width="1122" height="600" alt="image" src="https://github.com/user-attachments/assets/a04a147c-b63c-493b-8736-6845926f59fb" />

3. Once deployed, you should see a toast message on the bottom of the screen. Furthermore, the **Runtime Status** on top should show as **Started**.

<img width="1519" height="876" alt="image" src="https://github.com/user-attachments/assets/0f32a9e9-8f8c-4a03-b952-aae2ab13304b" />

4. You can also check the deployment status in the monitoring. Navigate to **Monitor > Integrations and APIs** from the menu, and select the tile **Manage Integration Content**.

<br>![image](/exercises/ex1/images/01_03_Deploy_04.png)

5. In the **Manage Integration Content** page, filter for your ID **XX**. You should see your deployed integration flow in status **Started**.

<img width="1755" height="823" alt="image" src="https://github.com/user-attachments/assets/ad2a27a4-cff6-4d76-a13e-b055fdef15fd" />

Now you are all set to test your scenario!

## Exercise 2.6 - Test and Monitor

To test your configuration scenario, we use the Bruno API client application for which we have provided a collection with pre-configured sample requests. As a prerequisite to test your integration scenario using the Bruno API client, you should have gone through [Setup Bruno API client](../ex0#setup-bruno-api-client/). If not, do the setup, then come back and proceed with the steps below.

1. Open the Bruno application on your laptop, expand the **EOIO Hands-on Workshop** collection and the **EOIO via Partitioned Queue** folder. Select the **Post Order in Sequence to AEM** POST request. Ensure that the right environment is selected which defines the host name of the tenant, the client id and the secret. Depending on which tenant you use in the exercises, select either **eu-02a** or **eu-02b**. In the provided URL, the **participant** variable should hold your user number <b>XX</b> assigned to you assuming that you have properly configured the parameter in the Bruno API client setup. The message GUID passed to the integration flow is automatically generated. Trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code **202 Accepted**. In our first test we leave the QueueID equals **12345**, so the Mapping does not get executed.

<img width="2559" height="974" alt="image" src="https://github.com/user-attachments/assets/179c053c-741c-4639-a55e-e8aea6c3c848" />

2. Navigate back to the monitoring page of Cloud Integration, and select the link **Monitor Message Processing**

<img width="1346" height="735" alt="image" src="https://github.com/user-attachments/assets/658da656-c994-4ff5-92c9-2a79eb6ed553" />

Then Filter for your Integration Flow.

<img width="1962" height="308" alt="image" src="https://github.com/user-attachments/assets/da7ea13d-3dcd-44d1-ba9e-5c87a75b962c" />

You can also filter based on the QueueID ...

<img width="2533" height="890" alt="image" src="https://github.com/user-attachments/assets/5b380d09-63df-4888-b104-5541f31c6d0a" />

... or the purchaseOrder (Custom Header = OrderID).

<img width="2500" height="937" alt="image" src="https://github.com/user-attachments/assets/a41fee96-744c-4f58-81e8-ff8ba50864a8" />

You should see 2 messages here: the first (bottom) is the message from the client to AEM, the second (top) is the Consumer from AEM to XI. You can identify it based on the Sender and Receiver Header.

<img width="1970" height="666" alt="image" src="https://github.com/user-attachments/assets/0eae798f-687c-4fe9-884c-4b0be15e620e" />

Our Mapping is only triggered when QueueID **not** equals **12345**. So let´s send another 2 Messages from Bruno with QueueID equals **123456** and 2 more with QueueID equals **67891** 

<img width="2138" height="514" alt="image" src="https://github.com/user-attachments/assets/1a18236f-fd16-480f-9d38-c3715fd7a4a7" />

You should see now fore each QueueID two Message with Status **Completed**. These are the Messages From the Client to AEM. Then there should be 1 Retry Message. This means AEM will try to push the message again in certain periods (defined in the Queue in AEM Broker Management).

<img width="2250" height="967" alt="image" src="https://github.com/user-attachments/assets/a35eb534-e0c7-4062-b467-efd228a15dea" />


Let´s check in AIM (https://mr-connection-h91kb3o1b6w.messaging.solace.cloud:943/?_gl=1*qknv9f*_gcl_au*MTQ1MTI1NTAyNC4xNzU2OTAxMDU0LjU1OTA2MDg5Ni4xNzU3MzMyOTk3LjE3NTczMzMwMTc.*_ga*MTc3MjY3NjI1OS4xNzExNjMzNDUy*_ga_XZ3NWMM83E*czE3NTczNDEzMzMkbzI2NiRnMSR0MTc1NzM0MjcwMCRqMzkkbDAkaDA.#/msg-vpns/YWVtX2NvbW11bml0eWNlbnRyYWw=?token=YWJj.eyJhY2Nlc3NfdG9rZW4iOiAibWlzc2lvbi1jb250cm9sLW1hbmFnZXI6NjJlYWZzZTNmdTdrOHEyZjFkdmdvMXM1cjEifQ%3D%3D.eHl6&title=AEM_CommunityCentral&subtitle=aem_communitycentral) 

<img width="2028" height="648" alt="image" src="https://github.com/user-attachments/assets/0786d61d-4053-4400-8113-2b302aa09eae" />

We can see that AEM has distributed the Messages to Partition 2 and 6. You can also see more if you click one partition and on **Messages Queued**. 

<img width="2545" height="565" alt="image" src="https://github.com/user-attachments/assets/d82d05e4-d2f4-4dd1-8001-f25d914c025a" />

This means that these 2 Partitions are blocked until the error is resolved. If you send another Message with QueueID equals **12345**, it should go through, unless it is assigned to one of the blocked partitions. Let´s try it:

<img width="2138" height="501" alt="image" src="https://github.com/user-attachments/assets/92536e05-8e09-411c-9fb3-34825c266f04" />

We can see in the Message Monitoring that it is completed:
<img width="2172" height="429" alt="image" src="https://github.com/user-attachments/assets/0128c097-ac29-4497-b1eb-9da447a32c21" />

Now let´s deploy the Message Mapping again:

<img width="2307" height="492" alt="image" src="https://github.com/user-attachments/assets/4d73e5ea-f18c-4632-bd50-fac07bd94e3e" />

Now, after Refresh on AEM the Queues are clear ...

<img width="1554" height="703" alt="image" src="https://github.com/user-attachments/assets/90e7d703-0165-402f-90e2-1b6a85db5aff" />

... and messages with QueueID **123456** and **67891** are completed.

<img width="2257" height="870" alt="image" src="https://github.com/user-attachments/assets/38a150b2-e351-42fe-a1a3-3c612b427fa5" />


## Summary

Congratulations. You have successfully modelled and tested an Exactly Once In Order scenario using Partitioned Queues.
