# Exercise 1 - Exactly Once In Order Scenario using Exclusive Queues

In this exercise, we will model and run an integration flow supporting Exactly Once In Order delivery by using an **exclusive JMS queue**. This ensures the following:
- By persisting the messages in a JMS queue, Cloud Integration can carry out the retry of the message delivery in case of an error.
- By using an exclusive JMS queue, the message sequence can be preserved.

You can anticipate the following situation: A message fails and is retried automatically until the message is successfully delivered. As long as the failed message is in retry, all successor messages are kept on hold to ensure that they don't overtake the predecessor message.

In the exercise, you won't start from scratch, instead you will use a template so that you can focus on the Exactly Once In Order specific settings only. In our example integration flow, a message is sent to an **SAP RM sender adapter** and directly stored in a JMS queue to guarantee message retry in case of a message processing error. The SAP RM protocol extends the plain SOAP protocol to support Exactly Once and Exactly Once In Order delivery by providing SAP proprietary SOAP headers or query parameters. To support Exactly Once, a **message id** needs to be passed to the integration flow. For Exactly Once In Order delivery, you need to transfer a **queue id** to the integration flow. If a queue ID is present, the quality of service is implicitly determined as Exactly Once In Order.

The second integration process reads the message from the very same JMS queue and runs the actual integration logic, in our case a message mapping. Once the message mapping has been successfully carried out, the message is reliably exchanged with a receiver using the XI 3.0 protocol. The **XI adapter** in Cloud Integration doesn’t natively support Exactly Once In Order delivery. Instead, you need to select the **Handled by Integration Flow** delivery assurance to implement the same. If you use the Handled by Integration Flow delivery assurance setting, the XI receiver adapter doesn’t persist the outgoing message which is not needed here because the message is persisted and retried from the exclusive JMS queue anyway. Furthermore, the XI receiver adapter expects the headers **SapQualityOfService** and **SapQueueId** to be set within the integration flow. Those headers are actually configured by using the corresponding headers passed from the SAP RM sender adapter.

For an improved monitoring of the exchanged messages, we have configured the corresponding SAP headers and custom header properties. This is already part of the provided template.

In order to simulate the error situation, we will use a re-usable message mapping in the integration flow as global resource which we intentionally won't deploy in the first place. To resolve the error, we will simply deploy the message mapping artifact. The failed message and all successor messages which were on hold will be eventually successfully delivered after an automatic retry.

## Exercise 1.1 - Copy Provided Templates

In the following, you will copy the provided integration flow and message mapping templates to your package. As a prerequisite, you should have created an own package. If you haven't created an own integration package yet, navigate to [Create an Integration package](/exercises/ex0/#create-an-integration-package), create a new package and then return. Otherwise, proceed with the next steps.

1. Open the provided SAP Integration Suite tenant (see [System Access](/main/#system-access)), and navigate to **Design > Integrations and APIs**. From there, open the integration package **EOIO Hands-on Workshop - Template** by selecting the same.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_01.png)

2. For message mappings, the copy functionality is not supported. So, we need to download and then import the message mapping template. In the integration package **EOIO Hands-on Workshop - Template**, switch to the **Artifacts** tab. Select the entry **Download** from the **Actions** menu of the message mapping **MM EOIO - Template** to download the message mapping artifact to your download folder.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_02.png)
   
3. Next, we will copy the provided integration flow template. Select **Copy** from the **Actions** menu of the integration flow **EOIO Exclusive Queue - Template**.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_03.png)
   
4. In the upcoming dialog, click on **Select** to select a target integration package.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_04.png)
   
5. From the list of packages, select your beforehand created package, i.e., package **User XX** where **XX** is the number assigned to you.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_05.png)

6. In the **Copy "EOIO Exclusive Queue - Template"** dialog, maintain the name of the target integration flow by replacing **Template_copy** with your user number **XX**. Then select **Copy**.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_06.png)

7. In the upcoming **Success** dialog, select **Navigate** to navigate to your package.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_07.png)

8. You should see the copied integration flow in the list of artifacts within your package **User XX**. Next, we need to upload the message mapping artifact. Switch to **Edit** mode.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_08.png)

9. Select entry **Message Mapping** from the **Add** menu.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_09.png)

10. In the upcoming **Add Message Mapping** dialog, select the **Upload** radio button option and then **Browse**.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_10.png)

11. Navigate to your download folder and select the beforehand downloaded message mapping artifact. i.e., file **MM EOIO - Template.zip**. Select **Open**.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_11.png)

12. Change the name of the to be created message mapping artifact by replacing **Template** with your user number **XX**. Select **Add**.

<br>![image](/exercises/ex1/images/01_01_CopyTemplates_12.png)

## Exercise 1.2 - Model your Integration Flow

Now that you have copied the provided templates, we should be all set to enhance the copied integration flow model to ensure Exactly Once In Order delivery.
    
1.  In your package, select the copied integration flow **EOIO Exclusive Queue - XX** with **XX** the ID assigned to you to open the model designer.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_01.png)
    
2. The integration flow contains two integration processes: the **Integration Process: Provider flow** to store the incoming message in the JMS queue, and the **Integration Process: Consumer flow** to read the message from the very same JMS queue. In the integration flow designer, switch to **Edit** mode.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_02.png)

3. First, we will maintain the exclusive JMS queue connection to decouple the two integration processes. In the editor, select the message end event **End** of the upper integration process **Integration Process: Provider flow**, then drag ...

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_03.png)
  
4. ... and drop a connection to the Receiver.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_04.png)

5. In the upcoming dialog, select the Adapter Type **JMS**. 

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_05.png)
   
6. Select the JMS connection. In the Properties section of the JMS receiver adapter, switch to the **Processing** tab, and enter the externalized parameter **participant** into the **Queue Name** field. Note, you create or reuse externalized parameters by placing the parameter name between opening and closing double curly brackets.

```yaml
{{participant}}
```

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_06.png)
   
7. Furthermore, change the **Access Type** to **Exclusive** from the drop down menu.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_07.png)

8. Scroll down to the **Integration Process: Consumer flow**, and add a JMS sender connection between the Sender and the message start event. On the tab **Connection** of the JMS sender adapter, maintain the same externalized parameter **participant** that we used before for the JMS receiver adapter into the **Queue Name** field, change the **Access Type** to **Exclusive**, and **unselect** the **Exponential Backoff** flag. Latter makes testing of the flow easier.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_08.png)

9. Next, we need to add the message mapping. Click outside of the integration processes to be able to configure the integration flow components. From the integration flow configuration, switch to tab **References**. Within the references, switch to tab **Global**, and select Message Mapping from the **Add References** menu.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_09.png)

10. In the upcoming **References** dialog, select the beforehand added message mapping **MM EOIO - XX** and select **OK**.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_10.png)

11. The message mapping should be placed behind the Groovy script of the **Integration Process: Consumer flow**. Select the flow step **Add custom header props** and select the **Plus** icon of the quick menu to add a new flow step.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_11.png)

12. Select the entry **Message Mapping** from the **Add Flow Step**. If the **Message Mapping** entry is not listed in the **Recommended Steps** section, search for it and then select the same.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_12.png)

13. Select the added message mapping flow step, and switch to tab **Processing**. Then, select **Select**.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_13.png)

14. In the upcoming dialog, switch to tab **Global Resources**, and select your message mapping **MM_EOIO__XX**. Select **OK**.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_14.png)

15. Next, we need to pass the quality of service and a queue id to the XI receiver adapter. Select the message mapping flow step, and add a new flow step from the quick menu like before.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_15.png)

16. In the **Add Flow Step**, select the **Content Modifier** entry below **Transformation**.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_16.png)

17. Select the added **Content Modifier** flow step, and switch to the **Message Header** tab. Add two new headers as follows:

      |Name                 | Source Type | Source Value        |
      | ------------------- | ------------| ------------------- | 
      | SapQueueId          | Header      | SapPlainSoapQueueId |
      | SapQualityOfService | Header      | SapPlainSoapQoS     |

**Note**: The headers **SapPlainSoapQoS** and **SapPlainSoapQueueId** are passed to the integraiton flow via the SAP RM adapter.

For your convenience, you can copy the header values from here:

```yaml
SapQueueId
```
```yaml
SapPlainSoapQueueId
```
```yaml
SapQualityOfService
```
```yaml
SapPlainSoapQoS
```

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_17.png)

18. Next, we need to configure the connection between the message end event and the Receiver of adapter type **XI**. Select the connection and switch to the **Delivery Assurance** tab. Maintain the parameters as follows:

As **XI Message ID Determination**, select **Map** from the drop down list.

Maintain **Source for XI Message ID** as
```yaml
${header.SapMessageIdEx}
```

As **Quality Of Service**, keep **Handled by Integration Flow**.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_18.png)

19. Finally **Save** your changes and then **Cancel** so that you can proceed with configuring the flow.

<br>![image](/exercises/ex1/images/01_02_ModelIntegrationFlow_19.png)


## Exercise 1.3 - Configure and Deploy your Integration Flow

In the following, you will configure and deploy the beforehand modified integration flow.
    
1.  After having saved and canceled, you should see the **Configure** button on the upper right. Select **Configure**.

<br>![image](/exercises/ex1/images/01_03_Deploy_01.png)
    
2. In the configuration dialog, on tab **Sender**, maintain the value of the **participant** parameter. Replace **XX** of the preconfigured value **UserXX** with the number assigned to you. The value of the parameter is actually appended to the integration flow end point to ensure a unique end point deployed on the tenant. Furthermore, the value equals the JMS queue name. Then **Save** and **Deploy**.

<br>![image](/exercises/ex1/images/01_03_Deploy_02.png)

3. Once deployed, you should see a toast message on the bottom of the screen. Furthermore, the **Runtime Status** on top should show as **Started**.

<br>![image](/exercises/ex1/images/01_03_Deploy_03.png)

4. You can also check the deployment status in the monitoring. Navigate to **Monitor > Integrations and APIs** from the menu, and select the tile **Manage Integration Content**.

<br>![image](/exercises/ex1/images/01_03_Deploy_04.png)

5. In the **Manage Integration Content** page, filter for your ID **XX**. You should see your deployed integration flow in status **Started**.

<br>![image](/exercises/ex1/images/01_03_Deploy_05.png)

Now you are all set to test your scenario!

## Exercise 1.4 - Test and Monitor

To test your configuration scenario, we use the Bruno API client application for which we have provided a collection with pre-configured sample requests. As a prerequisite to test your integration scenario using the Bruno API client, you should have gone through [Setup Bruno API client](../ex0#setup-bruno-api-client/). If not, do the setup, then come back and proceed with the steps below.

1. Open the Bruno application on your laptop, expand the **EOIO Hands-on Workshop** collection and the **EOIO via Exclusive Queue** folder. Select the **Post Order in Sequence** POST request. Ensure that the right environment is selected which defines the host name of the tenant, the client id and the secret. Depending on which tenant you use in the exercises, select either **eu-02a** or **eu-02b**. In the provided URL, the **participant** variable should hold your user number <b>XX</b> assigned to you assuming that you have properly configured the parameter in the Bruno API client setup. The message GUID passed to the integration flow is automatically generated. Trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code **202 Accepted**.

<br>![](/exercises/ex1/images/01_04_Test_01.png)

2. Now, let's send another message. For this, increase the **PurchaseOrderNumber** in the **Body** of the request. Then trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code **202 Accepted**. If you like, you can send a couple of more messages.

<br>![](/exercises/ex1/images/01_04_Test_02.png)

3. Navigate back to the monitoring page of Cloud Integration, and select the link **Monitor Message Processing** from your deployed integration flow **EOIO Exclusive Queue - XX**.

<br>![](/exercises/ex1/images/01_04_Test_03.png)

4. In the **Monitor Message Processing**, you should see three new logs belonging to your participant ID (assuming that you have triggered two test messages, otherwise you may see more logs). Two logs are in status **Completed** with Sender **TestClient_UserXX** and Receiver **JMSXQ_UserXX**. Those are the messages which went through the provider flow and which were put into the exclusive queue. In the **Custom Headers** section of the log, you should see the value of the **QueueID** and the value of the **OrderID** passed from the test client. Furthermore, the Quality of Order **QoS** is automatically set to **ExactlyOnceInOrder**.

<br>![](/exercises/ex1/images/01_04_Test_04.png)

5. Select the second log in status **Completed**. As you can see from the **OrderID** value, this log belongs to the second message that you sent.

<br>![](/exercises/ex1/images/01_04_Test_05.png)

6. Select the third log which is in status **Retry**. This is the message which was read from the exclusive queue via the consumer flow and were we intentionally forced an error. Here, the Sender should be **JMSXQ_UserXX** and the Receiver **MockedXIReceiver**. As you can see from the **Error Details**, the message went into an error because the referenced message mapping artifact is not available. Note, the second message is in sort of hold status as long as the predecessor message hasn't been either successfully processed or canceled. For this reason, we only see one log entry for the consumer flow.

<br>![](/exercises/ex1/images/01_04_Test_06.png)

7. Let's fix the error by deploying the referenced message mapping artifact. Navigate back to your integration package **User XX** with **XX** the number assigned to you, and select the **Deploy** entry from the **Actions** menu of your message mapping **MM EOIO- XX**.

<br>![](/exercises/ex1/images/01_04_Test_07.png)

8. Once deployed, you should see a toast message on the bottom of the screen.

<br>![](/exercises/ex1/images/01_04_Test_08.png)

9. Navigate back to the **Monitor Message Processing** and **Refresh**. The log of your first order should have changed from Status **Retry** to **Completed**

<br>![](/exercises/ex1/images/01_04_Test_09.png)

10. Furthermore, you should see a new log in status **Completed** belonging to the second order sent.

<br>![](/exercises/ex1/images/01_04_Test_10.png)


## Summary

Congratulations. You have successfully modelled and tested an Exactly Once In Order scenario using Exclusive Queues.

Continue to - [Exercise 2 - Exactly Once In Order scenario using partitioned queues](../ex2/README.md)
