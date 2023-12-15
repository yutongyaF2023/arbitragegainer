# Extra Credit Task 2: Distributed Agents

In order to adapt our current MailboxProcessors to a distributed cloud environment, we made use of FSharp.CloudAgent because this solution allowed us to keep our MailboxProcessors intact.

## Steps
1. Install and import necessary modules:
    ```
    open FSharp.CloudAgent
    open FSharp.CloudAgent.Messaging
    open FSharp.CloudAgent.Connections
    ```

2. Wrap existing agent inside an initialization function:
    ```
    let distributedAgent (agentId: ActorKey) =
    MailboxProcessor.Start(fun inbox ->
        ...
    )
    ```
    Note that this is the only change required to the existing agent code.

3. Create a new ServiceBus connection (we used our existing ServiceBus). Be sure to create the new queue referred to here; this is the queue where messages will be sent to and read by the agents:
    ```
    let connStr = "Endpoint=sb://arbitragegainer.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=RX56IkxeBgdYjM6OoHXozGRw37tsUQrGk+ASbNEYcl0="
    
    let queueName = "agentqueue"
    
    let cloudConn = CloudConnection.WorkerCloudConnection(ServiceBusConnection connStr, Connections.Queue queueName)
    ```

4. Listen on the connection, and add code for CloudAgent to create a new agent when a message that needs processing is received (this is how the CloudAgents work). 
    ```
    ConnectionFactory.StartListening(cloudConn, distributedAgent >> BasicCloudAgent)
    ```
The second part of the arguments to StartListening (`activationAgent >> BasicCloudAgent`)


## Issues
- The build ended up failing due to compatibility issues of FSharp.CloudAgent, System.ServiceModel, and .NET 6.0, which meant we could not test our solution.