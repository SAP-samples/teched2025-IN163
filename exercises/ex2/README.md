# Exercise 2 - Exactly Once In Order Scenario using partitioned Queues

In this exercise, we will model and run an integration flow supporting Exactly Once In Order delivery by using an **partitioned Queue on SAP Integration Suite, Advanced Event Mesh (AEM)**. This ensures the following:
- By persisting the messages in an AEM queue, Cloud Integration can carry out the retry of the message delivery in case of an error.
- By using a queue, the message sequence can be preserved.
- By using a partitioned queue, we enable parallelization to a certain extend (depends on the number of partitions).

As we have seen in the previous Exercise 1, if an error happens in an **exclusive JMS Queue**, the message is retried automatically until the message is successfully delivered. As long as the failed message is in retry, **all successor messages** are kept on hold to ensure that they don't overtake the predecessor message. When using **partitioned AEM Queues**, the behavior in case of an erroneous message slightly differs from using **exclusive JMS Queues**: As long as the failed message is in retry, **all successor messages in the same partition** (that is, with the same Partition Key Hash) are kept on hold to ensure that they don't overtake the predecessor message.

In the exercise, you won't start from scratch, instead you will use a template so that you can focus on the Exactly Once In Order specific settings only. In our example integration flow, a message is sent to an **SAP RM sender adapter** and directly stored in a partitioned queue on AEM to guarantee message retry in case of a message processing error. The SAP RM protocol extends the plain SOAP protocol to support Exactly Once and Exactly Once In Order delivery by providing SAP proprietary SOAP headers or query parameters. To support Exactly Once, a **message id** needs to be passed to the integration flow. For Exactly Once In Order delivery, you need to transfer a **queue id** to the integration flow. If a queue ID is present, the quality of service is implicitly determined as Exactly Once In Order. 

The difference to Exercise 1 is that AEM is not running inside of the SAP Integration Suite tenant, so you need to provide **connection details**. Additionally to the Queue ID on AEM, you also need to provide the User Property **JMSXGroupID** to AEM as Key Value for the generation of the Partition Key Hash. Based on this, the messages are distributed across the queue partitions.

Because AEM offers a lot of configuration options for setting up the Queues, in contrast to exercise 1 the Queue is not just created by maintaining it in the channel. You actively have to open up the **AEM Broker Manager** and create the Queue explicitly. Furthermore, you need to predefine how many partitions and consumers are required. For ensuring EOIO, the most important parameter here is **Maximum Delivered Unacknowledged Messages per Flow** in the Advanced Settings. If the value of this parameter is not 1, EOIO can not be guaranteed. 

The second integration process reads the message from the very same AEM queue and runs the actual integration logic, in our case a message mapping, but only if the QueueID **not** equals **12345**. The routing condition is added here to simulate an error situation: if the message mapping is not deployed, the messages in the corresponding Queue Partition stuck until the mapping is deployed and the first message is processed. If you choose the QueueID **12345**, first of all the messages won't run the message mapping and hence won't run into an error in the first place. Secondly, the messages are most likely distributed to a different partition because of a different generated Partition Key Hash (based on the QueueID) and hence are not blocked.

Once the message mapping has been successfully carried out, the message is reliably exchanged with a receiver using the XI 3.0 protocol. The **XI adapter** in Cloud Integration doesn’t natively support Exactly Once In Order delivery. Instead, you need to select the **Handled by Integration Flow** delivery assurance to implement the same. If you use the Handled by Integration Flow delivery assurance setting, the XI receiver adapter doesn’t persist the outgoing message which is not needed here because the message is persisted and retried from the exclusive JMS queue anyway. Furthermore, the XI receiver adapter expects the headers **SapQualityOfService** and **SapQueueId** to be set within the integration flow. Those headers are actually configured by using the corresponding headers passed from the SAP RM sender adapter.

For an improved monitoring of the exchanged messages, we have configured the corresponding SAP headers and custom header properties. This is already part of the provided template.

In order to simulate the error situation, we will use a re-usable message mapping in the integration flow as global resource which we intentionally won't deploy in the first place. To resolve the error, we will simply deploy the message mapping artifact. The failed message and all successor messages which were on hold will be eventually successfully delivered after an automatic retry.

## Exercise 2.1 - Undeploy Message Mapping from Exercise 1

Since we use the same Message Mapping like in Exercise 1, this needs to be undeployed before starting with Exercise 2 to be able to force an error situation.

Navigate to **Monitor > Integrations and APIs > Manage Integration Content**. Then filter based on your user ID **XX**. Select the message mapping **MM EOIO - XX** and click **Undeploy** to undeploy your message mapping artifact.

<br>![image](/exercises/ex2/images/02_01_01.png)

## Exercise 2.2 - Copy Provided Template

In the following, you will copy the provided integration flow to your package. As you did this before in Exercise 1, we won't describe this in very detail here. Run through the similar steps like in Exercise 1.1. This time, you do not need to copy the message mapping template as we will reuse the message mapping from Exercise 1.

1. Open the package **EOIO Hands-on Workshop - Template** and copy the integration flow template **EOIO Partitioned Queue - Template** to your package **User XX** with **XX** the ID assigned to you.

<br>![image](/exercises/ex2/images/02_02_01.png)

2. As name for the copied integration flow, choose **EOIO Partitioned Queue - XX** with **XX** the ID assigned to you. Ensure that your target package **User XX** is selected.

<br>![image](/exercises/ex2/images/02_02_02.png)

## Exercise 2.3 - Model your Integration Flow

Now that you have copied the provided template, we should be all set to enhance the copied integration flow model to ensure Exactly Once In Order delivery.
    
1.  In your package, select the copied integration flow **EOIO Partitioned Queue - XX** with **XX** the ID assigned to you to open the model designer.

<br>![image](/exercises/ex2/images/02_03_01.png)
    
2. The integration flow contains two integration processes: the **Integration Process: Provider flow** to store the incoming message in the AEM queue, and the **Integration Process: Consumer flow** to read the message from the very same AEM queue. In the integration flow designer, switch to **Edit** mode.

<br>![image](/exercises/ex2/images/02_03_02.png)

3. First, we will maintain the AEM connection to decouple the two integration processes. In the editor, select the message end event **End** of the upper integration process **Integration Process: Provider flow**, then drag ...

<br>![image](/exercises/ex2/images/02_03_03.png)
  
4. ... and drop a connection to the Receiver.

<br>![image](/exercises/ex2/images/02_03_04.png)

5. In the upcoming dialog, select the Adapter Type **AdvancedEventMesh**. 

<br>![image](/exercises/ex2/images/02_03_05.png)

6. Select the AEM connection. In the **Properties** section of the AEM receiver adapter, click on **Externalize**.

<br>![image](/exercises/ex2/images/02_03_06.png)

7. On the **Externalization** dialog, stay on tab **Connection**, and maintain the Externalized Parameters as follows. When done, select **OK** to close the **Externalization** dialog.

|Name                   | Value                     |
| ----------------------| ---------------------------| 
| Host                  | **{{AEM_Host}}**                | 
| MessageVPN           | **{{AEM_MessageVPN}}**                     |
| Username              | **{{AEM_Username}}**                      |
| PasswordSecureAlias | **{{AEM_PasswordSecureAlias}}**                                               |

<br>![image](/exercises/ex2/images/02_03_07.png)
   
8. Switch to the **Processing** tab. Maintain the Parameters as follows:

| Name              | Value                       |
| ----------------- | --------------------------- |
| Delivery Mode     | **Persistent (Guaranteed)**     |
| Endpoint Type     | **Queue**                       |
| Destination Name  | **HOW_EOIO_PQ_{{participant}}** |

Note that here we externalize parts of the parameters so we can only assign one value and use them in other parts as well (e.g. in the Receiver and Sender Channel of the AEM Connection)

<br>![image](/exercises/ex2/images/02_03_08.png)

9. Switch to the **Message Properties** tab and add the following Properties. The properties are stored on the AEM queue so that they can be used in the consumer integration flow.

|Name                   | Value                       |
| ----------------------| ----------------------------|
| Application Message ID         | **${header.SapMessageIdEx}**    |
| Sender ID         | **${header.SAP_Receiver}**    |

Scroll down to the **User Properties** section, add the **JMSXGroupID** and **purchaseOrder** properties. As mentioned above, the property **JMSXGroupID** is used to create the partition hash value to distribute the message to the corresponding partition.

|Key            | Value                         |    
| --------------| ----------------------------|
| JMSXGroupID    | **${header.SapPlainSoapQueueId}** |
| purchaseOrder    | **${header.purchaseOrder}** |
   
<br>![image](/exercises/ex2/images/02_03_09.png)

10. In the integration flow model, scroll down to the **Integration Process: Consumer flow**, and add an AEM sender connection between the Sender and the message start event in a similar way how you did this for the AEM receiver adapter by dropping and dragging a connection between the sender and the message start event. Now, let's maintain the very same Externalized Parameters for the sender connection as well. There is another option how to do this actually without entering the Externalization dialog. You can stay in the **Properties** section of the AEM sender adapter and switch to the **Connection** tab. Here, maintain the values as follows. For the **Username**, ensure to overwrite the **default** value.

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | **{{AEM_Host}}**  | 
| MessageVPN           | **{{AEM_MessageVPN}}**                                          |
| Username              | **{{AEM_Username}}**                                         |
| PasswordSecureAlias | **{{AEM_PasswordSecureAlias}}**                                               |

<br>![image](/exercises/ex2/images/02_03_10.png)
  
11. Switch to the **Processing** tab, and maintain the following parameters:

| Name              | Value                       |
| ----------------- | --------------------------- |
| Run on a single worker node     | Select the check box    |
| Parallel consumers     | **10**                       |
| Queue Name  | **HOW_EOIO_PQ_{{participant}}** |
| Acknowledgement Mode  | **Automatic On Exchange Complete** |
| Settlement Outcome After maximum Attempts  | Keep the default which is **Failed** |

<br>![image](/exercises/ex2/images/02_03_11.png)

12. Next, we need to add the message mapping. Click outside of the integration processes to be able to configure the integration flow components. From the integration flow configuration, switch to tab **References**. Within the references, switch to tab **Global**, and select **Message Mapping** from the **Add References** menu.

<br>![image](/exercises/ex2/images/02_03_12.png)

13. In the upcoming **References** dialog, select the beforehand added message mapping **MM EOIO - XX** and select **OK**.

<br>![image](/exercises/ex2/images/02_03_13.png)

14. The message mapping should be assigned behind the Router condition in the Step **Message Mapping** in the **Integration Process: Consumer flow**. Select the flow step **Message Mapping**, select the **Processing** tab and click **Select**.

<br>![image](/exercises/ex2/images/02_03_14.png)

15. Choose the **Global Ressources** tab and select the beforehand added Message Mapping **MM_EOIO_XX** with **XX** the user ID assigned to you. Select **OK**.

<br>![image](/exercises/ex2/images/02_03_15.png)

16. **Save** your Integration Flow.

<br>![image](/exercises/ex2/images/02_03_16.png)

## Exercise 2.4 - Configure Queue on AEM Broker

1. Open the AEM broker (see [System Access](/#system-access)), navigate to the **Cluster Manager**, and select the broker **teched-2025-in163**.

<br>![image](/exercises/ex2/images/02_04_aem_new_02.png)

2. In the **Service Details** page of the broker, select the **Open Broker Manager** link from the upper right.

<br>![image](/exercises/ex2/images/02_04_aem_new_03.png)

3. In the Broker Manager, select the **Queues** entry from the navigation pane, and select the **+ Queue** button to create a new queue.

<br>![image](/exercises/ex2/images/02_04_aem_new_04.png)

4. In the upcoming dialog, enter the Queue Name **HOW_EOIO_PQ_UserXX** with **XX** the user ID assigned to you. Click **Create**.

<br>![image](/exercises/ex2/images/02_04_aem_new_05.png)

5. In the **Edit Queue Settings**, change the Access Type to **Non-Exclusive** and the **Partition Count** as well as the **Maximum Consumer Count** to **10**. Then select the **Show Advanced Settings** link.

<br>![image](/exercises/ex2/images/02_04_aem_new_06.png)
   
6. Scroll down, and change the parameter **Maximum Delivered Unacknowledged Messages per Flow** to **1** (needed for ensuring EOIO). Then, **Apply** your configuration.

<br>![image](/exercises/ex2/images/02_04_aem_new_07.png)

7. A new queue should show up in the **Queues** tab.

<br>![image](/exercises/ex2/images/02_04_aem_new_08.png)

## Exercise 2.5 - Configure and Deploy your Integration Flow

In the following, you will configure and deploy the beforehand modified integration flow.
    
1.  Navigate back to your integration flow, and select the **Configure** button on the upper right. If you can't see the button, you may first have to **Cancel** assuming that you have first saved your changes. Select **Configure**.

<br>![image](/exercises/ex2/images/02_05_01.png)

2.  In the upcoming dialog, stay on tab **Sender** and select the Sender **AEM_Sender** from the drop down menu. Set up the Connection Details for AEM and maintain your participant number as follows. When done, **Save** and **Deploy**.

|Name                   | Value                                                         |
| ----------------------| --------------------------------------------------------------| 
| Host                  | **tcps://mr-connection-6jekt6djxiu.messaging.solace.cloud:55443** | 
| Message VPN           | **teched-2025-in163**                                          |
| Username              | **solace-cloud-client**                                           |
| Password Secure Alias | **AEM**                                               |
| participant           | **UserXX** with **XX** your user id                                   |

**Note**: Because we use externalized parameters, the connection details for the AEM_Receiver adapter is automatically set.

<br>![image](/exercises/ex2/images/02_05_02.png)

3. Once deployed, you should see a toast message on the bottom of the screen. Furthermore, the **Runtime Status** on top should show as **Started**. You can also switch to tab **Deployment Status** to gather more details about the deployment status such as the deployed version.

<br>![image](/exercises/ex2/images/02_05_03.png)

4. You can also check the deployment status in the monitoring. Navigate to **Monitor > Integrations and APIs** from the menu, and select the tile **Manage Integration Content**.

<br>![image](/exercises/ex2/images/02_05_04.png)

5. In the **Manage Integration Content** page, filter for your ID **XX**. You should see your deployed integration flow in status **Started**.

<br>![image](/exercises/ex2/images/02_05_05.png)

Now you are all set to test your scenario!

## Exercise 2.6 - Test and Monitor

To test your configuration scenario, we use the Bruno API client application for which we have provided a collection with pre-configured sample requests. As a prerequisite to test your integration scenario using the Bruno API client, you should have gone through [Setup Bruno API client](../ex0#setup-bruno-api-client/). If not, do the setup, then come back and proceed with the steps below.

1. Open the Bruno application on your laptop, expand the **EOIO Hands-on Workshop** collection and the **EOIO via Partitioned Queue** folder. Select the **Post Order in Sequence to AEM** POST request. Ensure that the right environment is selected which defines the host name of the tenant, the client id and the secret. Depending on which tenant you use in the exercises, select either **eu-02a** or **eu-02b**. In the provided URL, the **participant** variable should hold your user number <b>XX</b> assigned to you assuming that you have properly configured the parameter in the Bruno API client setup. The message GUID passed to the integration flow is automatically generated. Trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code **202 Accepted**. In our first test we leave the QueueID equals **12345**, so the Mapping does not get executed.

<br>![image](/exercises/ex2/images/02_06_01.png)

2. Navigate back to the **Manage Integration Content** page of Cloud Integration. From your deployed integration flow **EOIO Partitioned Queue - XX**, scroll down to the **Artifacts Details** and select the link **Monitor Message Processing** to open the message monitor.

<br>![image](/exercises/ex2/images/02_06_03.png)

3. This ensures that the message monitor is already filtered for your Integration Flow. In addition, you can filter based on the custom header properties, like for the **OrderID**.

<br>![image](/exercises/ex2/images/02_06_05.png)

4. You can also filter based on the **QueueID**. You should see 2 messages here: the first (bottom) is the message from the client to AEM, the second (top) is the Consumer from AEM to XI. You can identify it based on the Sender and Receiver Header.

<br>![image](/exercises/ex2/images/02_06_04.png)

5. Our Mapping is only triggered when QueueID **not** equals **12345**. So let´s navigate back to the Bruno test client. Send another message from Bruno with QueueID equals **123456**. Also increase the purchase order number in the body of the message. Then click the **Send** button.

<br>![image](/exercises/ex2/images/02_06_06.png)

6. Send another 2 messages from Bruno with QueueID equals **67891**. Click the **Send** button twice but before increase the purchase order number in the body of the message for each of the messages.

<br>![image](/exercises/ex2/images/02_06_07.png)

7. Navigate back to the message monitor and filter for the QueueID equals **67891**. You should see now two messages in status **Completed**. These are the messages from the Client to AEM. Then there should be 1 message in status **Retry**. This means, AEM will try to push the message again in certain periods (defined in the Queue in AEM Broker Management).

<br>![image](/exercises/ex2/images/02_06_08.png)

8. Let´s navigate back to the AEM broker. In the **Queues** monitor, filter for your user ID **UserXX**, and select your queue. 

<br>![image](/exercises/ex2/images/02_06_aem_01.png)

9. Switch to tab **Partitions**. We can see that AEM has distributed the Messages to Partition 2 and 6. Partition 2 has 2 messages, partition 6 one. Depending on which **QueueID** value you have chosen, the partition numbers for your tests may differ. You can also see more details if you click one partition and there on **Messages Queued**. So, those two partitions are blocked until the error has been resolved.

<br>![image](/exercises/ex2/images/02_06_aem_02.png)

10. If you send another Message with QueueID equals **12345**, it should go through, unless it is assigned to one of the blocked partitions. Let´s try it.

<br>![image](/exercises/ex2/images/02_06_09.png)

11. We can see in the Message Monitoring that it is completed.

<br>![image](/exercises/ex2/images/02_06_10.png)

12. Now let´s fix the error by deploying the message mapping again. Open your package **User XX**, and deploy your message mapping **MM EOIO - XX**.

<br>![image](/exercises/ex2/images/02_06_11.png)

13. Now, after Refresh on AEM the Queues are clear ...

<br>![image](/exercises/ex2/images/02_06_aem_03.png)

14. ... and messages with QueueID **123456** and **67891** are completed.

<br>![image](/exercises/ex2/images/02_06_12.png)


## Summary

Congratulations. You have successfully modelled and tested an Exactly Once In Order scenario using Partitioned Queues.
