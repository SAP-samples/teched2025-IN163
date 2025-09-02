# Exercise 1 - Exactly Once In Order scenario using exclusive queues

In this exercise, we will model and run an integration flow supporting Exactly Once In Order delivery by using an **exclusive JMS queue**. This ensures the following:
- By persisting the messages in a JMS queue, Cloud Integration can carry out the retry of the message delivery in case of an error.
- By using an exclusive JMS queue, the message sequence can be preserved.

You can anticipate the following situation: A message fails and is retried automatically until the message is successfully delivered. As long as the failed message is in retry, all successor messages are kept on hold to ensure that they don't overtake the predecessor message.

In the exercise, you won't start from scratch instead you will use a template so that you can focus on the Exactly Once In Order specific settings only. In our example integration flow, a message is sent to an SAP RM sender adapter and directly stored in a JMS queue to guarantee message retry in case of a message processing error. The SAP RM protocol extends the plain SOAP protocol to support Exactly Once and Exactly Once In Order delivery by providing SAP proprietary SOAP headers or query parameters. To support Exactly Once, a **message id** needs to be passed to the integration flow. For Exactly Once In Order delivery, you need to transfer a **queue id** to the integration flow. If a queue ID is present, the quality of service is implicitly determined as Exactly Once In Order.

The second integration process reads the message from the very same JMS queue and runs the actual integration logic, in our case a message mapping. Once the message mapping has been successfully carried out, the message is reliably exchanged with a receiver using the XI 3.0 protocol. The XI adapter in Cloud Integration doesn’t natively support Exactly Once In Order delivery. Instead, you need to select the **Handled by Integration Flow** delivery assurance to implement the same. If you use the Handled by Integration Flow delivery assurance setting, the XI receiver adapter doesn’t persist the outgoing message which is not needed here because the message is persisted and retried from the exclusive JMS queue anyway. Furthermore, the XI receiver adapter expects the headers **SapQualityOfService** and **SapQueueId** to be set within the integration flow. Those headers are actually configured by using the corresponding headers passed from the SAP RM sender adapter.

For an improved monitoring of the echanged messages, we have configured the corresponding SAP headers and custom header properties. This is already part of the provided template.

In order to simulate the error situation, we will use a re-usable message mapping in the integration flow as global resource which we intentionally won't deploy in the first place. To resolve the error, we will simply deploy the message mapping artifact. The failed message and all successor messages which were on hold will be eventually successfully delivered after an automatic retry.

## Copy provided integration flow

In the following, you will copy the integration flow **Pattern Quality Of Service - EOIO Template** provided to you.

1. Open your previously created package, and switch to the <b>Artifacts</b> tab, then switch to <b>Edit</b> Mode. If you haven't created an own integration package yet, navigate to [Create an Integration package](/exercises/ex0/#create-an-integration-package), create a new package and then return. Otherwise, proceed with the next steps.

<br>![](/exercises/ex2/images/02_01_Migrate_01.png)

1. Navigate back to the list of packages by clicking the **Integrations and APIs** bread crumb.

<br>![image](/exercises/ex2/images/02-00-NavigateBack.png)

2. Open the package **Exactly Once & Exactly Once In Order Use Cases** by selecting the same.

<br>![image](/exercises/ex2/images/02-01-EnterEOPackage.png)
   
3. In the package **Exactly Once & Exactly Once In Order Use Cases**, switch to tab **Artifacts** and select **Copy** from the **Actions** menu of the integration flow **Pattern Quality Of Service - EOIO Template**.

<br>![](/exercises/ex4/images/04-01-Copy.png)
   
4. In the upcoming dialog, click on **Select** to select a target integration package.

<br>![](/exercises/ex4/images/04-02-Select.png)
   
5. From the list of packages, select your beforehand created package, i.e., package **User XX** where **XX** is the number assigned to you.

<br>![](/exercises/ex4/images/04-03-SelectPackageXX.png)

6. In the **Copy "Pattern Quality Of Service - EOIO Template"** dialog, maintain the name of the target integration flow by replacing **Template_copy** with your user number **XX**. Then select **Copy**.

<br>![image](/exercises/ex4/images/04-04-Copy.png)

7. In the upcoming **Success** dialog, select **Navigate** to navigate to your package.

<br>![](/exercises/ex4/images/04-05-Navigate.png)


## Model your integration flow

In the following, you will enhance the copied integration flow model to ensure Exactly Once In Order delivery.
    
1.  In your package, select the copied integration flow **Pattern Quality Of Service - EOIO XX** with **XX** the ID assigned to you to open the model designer.

<br>![](/exercises/ex4/images/04-06-OpenFlow.png)
    
2. In the integration flow designer, switch to **Edit** mode.

<br>![](/exercises/ex4/images/04-07-EditMode.png)

3. In the editor, select the **Send** step of the **Integration Process: Provider flow**, then drag ...

<br>![](/exercises/ex4/images/04-08-SelectConnector.png)
  
4. ... and drop a connection to the Receiver.

<br>![](/exercises/ex4/images/04-09-DropConnector.png)

5. In the upcoming dialog, select the Adapter Type **JMS**. 

<br>![](/exercises/ex4/images/04-10-SelectJMS.png)
   
6. In the Properties section of the JMS receiver adapter, switch to the **Processing** tab, and enter the externalized parameter **queueName** into the **Queue Name** field. Note, you create or reuse externalized parameters by placing the parameter name between opening and closing double curly brackets.

<br>Queue Name =
```yaml
{{queueName}}
```

<br>![](/exercises/ex4/images/04-11-EnterExternalized.png)
   
7. Furthermore, change the **Access Type** to **Exclusive** from the drop down menu.

<br>![](/exercises/ex4/images/04-12-SelectExclusive.png)

8. Scroll down to the **Integration Process: Consumer flow**, and add a JMS sender connection between the Sender and the message start event. On the tab **Connection** of the JMS sender adapter, maintain the same externalized parameter **queueName** that we used before for the JMS receiver adapter into the **Queue Name** field, change the **Access Type** to **Exclusive**, and **unselect** the **Exponential Backoff** flag. Latter makes testing of the flow easier.

<br>![](/exercises/ex4/images/04-14-JMSSender.png)

9. Next, we need to pass the quality of service and a queue id to the XI receiver adapter. From the editor palette, select the **Content Modifier** entry below the **Transformation** menue and drag ...

<br>![](/exercises/ex4/images/04-16a-SelectContentModifier.png)

10. ... and drop the flow step on the connecting line between the router and the message end event. In the **Content Modifier**, switch to the **Message Header** tab, and add two new headers as follows:

Create header with Name **SapQualityOfService**, Source Type **Constant**, and value **ExactlyOnceInOrder**

Create header with Name **SapQueueId**, Source Type **Header**, and value **queueid**

<br>![](/exercises/ex4/images/04-17-AddHeaders.png)

11. Maintain the connection between the message end event and the Receiver of adapter type **XI**. Switch to the **Delivery Assurance** tab, and maintain the parameters as follows:

As **XI Message ID Determination**, select **Map** from the drop down list.

Maintain **Source for XI Message ID** as
```yaml
${header.messageid}
```

As **Quality Of Service**, select **Handled by Integration Flow** from the drop down menu.

<br>![](/exercises/ex4/images/04-18-XIAdapter.png)

12. Finally **Save** your changes and then **Cancel** so that you can proceed with configuring the flow.

<br>![](/exercises/ex4/images/04-19-Save.png)


## Configure and deploy your integration flow

In the following, you will configure and deploy the beforehand modified integration flow.
    
1.  After having saved and canceled, you should see the **Configure** button on the upper right. Select **Configure**.

<br>![](/exercises/ex4/images/04-20-Configure.png)
    
2. In the configuration dialog, maintain the value of the **participantNumber** parameter. Replace **XX** with the number assigned to you. The parameter value is actually appended to the integration flow end point to ensure a unique end point deployed on the tenant.

<br>![](/exercises/ex4/images/04-21-ConfigureXX.png)

3. Select the **Sender1** from the Sender drop down, and maintain the value of the **Queue Name** parameter. Replace **XX** of the preconfigured queue name **eoie_XX** with the number assigned to you. Then **Save** and **Deploy**.

<br>![](/exercises/ex4/images/04-22-SaveAndDeploy.png)
  
4. Once deployed, you should see a toast message on the bottom of the screen. Furthermore, the **Runtime Status** on top should show as **Started**.

<br>![](/exercises/ex4/images/04-23-Started.png)

5. Let's fetch the end point of the deployed integration flow. Navigate to **Monitor > Integrations and APIs** from the menu, and select the tile **Manage Integration Content**.

<br>![](/exercises/ex4/images/04-23b-ManageIntegrationContent.png)

6. In the **Manage Integration Content** page, filter for your ID **XX**. You should see your deployed integration flow in status **Started**. Select the **Copy entry point URL to clipboard** button next to the integration flow's end point as we will use it in the next step.

<br>![](/exercises/ex4/images/04-23c-CopyEndPoint.png)

Now you are all set to test your scenario!

## Testing

In this chapter, you will test the integration flow. In the following, Bruno is used. However you may use any other http client tool.

**Note**: You have two options to execute and test your integration scenario:
- The quickest option is to use the Bruno API client application for which we have provided a collection with pre-configured sample request. As a prerequisite to test your integration scenario using the Bruno API client, you should have gone through [Setup Bruno API client](../prep/). If not, do the setup, then come back and proceed with [option 1](#option-1-using-bruno-api-client).
- If you like to use your own tool, we have described in detail how to setup a sample request incl. body and authentication. This is described in [option 2](#option-2-using-your-own-api-client).

### Option 1: Using Bruno API client

1. Open the Bruno application on your laptop, expand the **Guidelines Exercises** collection and select the POST request **EOIO request**. Paste the copied end point from the clipboard into the URL field or simply replace the **XX** in the URL with the id provided to you. First, we like to create a message which should intentionally result into an error to block the queue. Switch to the **Headers** tab, and change the value of the header **forceError** to **true**. Ensure that the **eu03** environment has been selected. Then trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code 200 and a response confirming that the message has been passed to the exclusive queue.

<br>![](/exercises/ex4/images/04-25-BrunoError.png)

2. Now, let's send another message which should be at the end successfully processed. Change the value of the header **forceError** back to **false**. Then trigger a message by selecting the **Send Request** button on the upper right. The request should return HTTP code 200 and a response confirming that the message has been passed to the exclusive queue.

<br>![](/exercises/ex4/images/04-26-BrunoSuccess.png)


### Option 2: Using your own API client

1. Open your own API client and create a new **POST** request.

2. Paste the copied end point from the clipboard into the URL field. 

3. Enter the following payload into the body section of the sample message.

```xml
<?xml version="1.0"?>
<ns0:PurchaseOrder xmlns:ns0="http://demo.sap.com/eip/qos" PurchaseOrderNumber="XX-101" OrderDate="2021-10-05">
	<DeliveryNotes>This is a test message for EO</DeliveryNotes>
	<Items>
		<Item ItemNumber="10">
			<ProductId>HT-1000</ProductId>
			<ProductName>Notebook Basic 15</ProductName>
			<Category>Notebooks</Category>
			<Quantity>1</Quantity>
			<CurrencyCode>EUR</CurrencyCode>
			<Price>956.00</Price>
		</Item>
		<Item ItemNumber="20">
			<ProductId>HT-1001</ProductId>
			<ProductName>Notebook Basic 17</ProductName>
			<Category>Notebooks</Category>
			<Quantity>1</Quantity>
			<CurrencyCode>EUR</CurrencyCode>
			<Price>1249.00</Price>
		</Item>
		<Item ItemNumber="30">
			<ProductId>HT-1030</ProductId>
			<ProductName>Ergo Screen</ProductName>
			<Category>Flat screens</Category>
			<Quantity>2</Quantity>
			<CurrencyCode>EUR</CurrencyCode>
			<Price>460.00</Price>
		</Item>
	</Items>
</ns0:PurchaseOrder>
```

4. To authenticate to the Cloud Integration runtime, select **Basic Authentication** and maintain the credentials as follows.

<br>USERNAME =
```yaml
sb-3009327f-3dc1-4e3e-9853-5bd7c23e221d!b44358|it-rt-cpisuite-europe-03!b18631
```
<br>PASSWORD = 
```yaml 
e507568e-892c-443f-a6ba-4d53f76fecac$wS5Kq2nV25PlNT-U8bh8Yd-HGoBZpO-XW7Za9X3URE0=
```

5. Add the following headers:
- **forceError** with value **true**
- **queueid** with any value
- **messageid** with a GUID as hexadecimal digits.

Note, in Bruno we need to put the following script into the Pre Request to generate a v4 GUID. Then  we can use the variable **{{uuid}}** in the header.

```yaml
bru.setVar('uuid', require("uuid").v4());
```

In Postman, you can simply use **{{$guid}}**.

6. Trigger a message. Upon success, you should get a response confirming that the message has been passed to the exclusive queue.
7. Change the **forceError** header to **false**, then resend the message. You should get a response confirming that the message has been passed to the exclusive queue.

## Monitor the messages and queue

In the following, we monitor the processed messages to confirm that In Order processing is kept.

1. Navigate back to the monitoring page of Cloud Integration, and select the tile **Monitor Message Processing**.

<br>![image](/exercises/ex4/images/04-27-MonitorTile.png)

2. In the **Monitor Message Processing**, you should see two logs belonging to your participant ID in status **Completed** with Sender **XX** and Receiver **JMSExclusiveQueue**. Those are the messages which went through the provider flow and which were put into the exclusive queue.

<br>![image](/exercises/ex4/images/04-28-MPLProvider.png)

3. One more message log is in status **Retry**. This is the message which was read from the exclusive queue via the consumer flow and were we intentionally forced an error. Note, the second message is in sort of hold status as long as the predecessor message hasn't been either successfully processed or canceled. For this reason, we only see one log entry for the consumer flow.

<br>![image](/exercises/ex4/images/04-29-MPLConsumer.png)

4. Let's monitor the queues. Navigate back to the monitoring overview page, and select the tile **Message Queues**.

<br>![image](/exercises/ex4/images/04-30-QueueTile.png)

5. In the **Manage Message Queues**, filter for your participant number, and select the exclusive queue **eoio_XX** with **XX** the participant numnber assigned to you. You should see two entries in **Waiting** status. The first actually belongs to the erroneous message which is in retry, the second belongs to the successor message which is on hold. Since we always trigger an error whenever the first message is retried, the only option to resolve the situation is to delete the message from the exclusive queue. Select the first entry and then select **Delete**.

<br>![image](/exercises/ex4/images/04-31-QueueDeleteEntry.png)

6. In the upcoming dialog, you need to confirm the deletion.

<br>![image](/exercises/ex4/images/04-32-ConfirmDelete.png)

7. Once the message was deleted from the queue, keep refreshing. After a while, the second message should be removed from the queue and hopefully succesfully processed.

<br>![image](/exercises/ex4/images/04-33-Refresh.png)

8. Navigate back to the **Monitor Message Processing**. You should see one log in status **Cancelled** and a new log of your integration flow in status **Completed**. 

<br>![image](/exercises/ex4/images/04-34-MPLFinal.png)


## Summary

Congratulations. You have successfully modelled and tested an Exactly Once In Order scenario.

Continue to - [Exercise 2 - Exactly Once In Order scenario using partitioned queues](../ex2/README.md)
