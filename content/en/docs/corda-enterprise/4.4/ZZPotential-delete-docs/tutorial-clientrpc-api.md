---
title: "Using the client RPC API"
date: 2020-01-08T09:59:25Z
---



# Using the client RPC API
In this tutorial we will build a simple command line utility that connects to a node, creates some cash transactions
            and dumps the transaction graph to the standard output. We will then put some simple visualisation on top. For an
            explanation on how RPC works in Corda see clientrpc.

We start off by connecting to the node itself. For the purposes of the tutorial we will use the Driver to start up a notary
            and a Alice node that can issue, move and exit cash.

Here’s how we configure the node to create a user that has the permissions to start the `CashIssueFlow`,
            `CashPaymentFlow`, and `CashExitFlow`:

Now we can connect to the node itself using a valid RPC user login and start generating transactions in a different
            thread using `generateTransactions` (to be defined later):

`proxy` exposes the full RPC interface of the node:

The RPC operation we need in order to dump the transaction graph is `internalVerifiedTransactionsFeed`. The type
            signature tells us that the RPC operation will return a list of transactions and an `Observable` stream. This is a
            general pattern, we query some data and the node will return the current snapshot and future updates done to it.
            Observables are described in further detail in clientrpc

The graph will be defined as follows:


* Each transaction is a vertex, represented by printing `NODE <txhash>`


* Each input-output relationship is an edge, represented by printing `EDGE <txhash> <txhash>`


Now we just need to create the transactions themselves!

We utilise several RPC functions here to query things like the notaries in the node cluster or our own vault. These RPC
            functions also return `Observable` objects so that the node can send us updated values. However, we don’t need updates
            here and so we mark these observables as `notUsed` (as a rule, you should always either subscribe to an `Observable`
            or mark it as not used. Failing to do so will leak resources in the node).

Then in a loop we generate randomly either an Issue, a Pay or an Exit transaction.

The RPC we need to initiate a cash transaction is `startFlow` which starts an arbitrary flow given sufficient
            permissions to do so.

Finally we have everything in place: we start a couple of nodes, connect to them, and start creating transactions while
            listening on successfully created ones, which are dumped to the console. We just need to run it!

```text
# Build the example
./gradlew docs/source/example-code:installDist
# Start it
./docs/source/example-code/build/install/docs/source/example-code/bin/client-rpc-tutorial Print
```
Now let’s try to visualise the transaction graph. We will use a graph drawing library called [graphstream](http://graphstream-project.org/).

If we run the client with `Visualise` we should see a simple random graph being drawn as new transactions are being created.


## Whitelisting classes from your CorDapp with the Corda node
As described in clientrpc, you have to whitelist any additional classes you add that are needed in RPC
                requests or responses with the Corda node.  Here’s an example of both ways you can do this for a couple of example classes.

See more on plugins in running-a-node.


## Security
RPC credentials associated with a Client must match the permission set configured on the server node.
                This refers to both authentication (username and password) and authorisation (a permissioned set of RPC operations an
                authenticated user is entitled to run).

In the instructions below the server node permissions are configured programmatically in the driver code:

```text
driver(driverDirectory = baseDirectory) {
    val user = User("user", "password", permissions = setOf(startFlow<CashFlow>()))
    val node = startNode("CN=Alice Corp,O=Alice Corp,L=London,C=GB", rpcUsers = listOf(user)).get()
```
When starting a standalone node using a configuration file we must supply the RPC credentials in the node configuration,
                like illustrated in the sample configuration below, or indicate an external database storing user accounts (see
                clientrpc for more details):

```text
rpcUsers : [
    { username=user, password=password, permissions=[ StartFlow.net.corda.finance.flows.CashFlow ] }
]
```
Wildcard permissions can be set by using the *** character, e.g.:

```text
rpcUsers : [
    { username=user, password=password, permissions=[ StartFlow.net.corda.finance.flows.* ] }
]
```
When using the gradle Cordformation plugin to configure and deploy a node you must supply the RPC credentials in a similar
                manner:

```text
rpcUsers = [
        ['username' : "user",
         'password' : "password",
         'permissions' : ["StartFlow.net.corda.finance.flows.CashFlow"]]
]
```
You can then deploy and launch the nodes (Notary and Alice) as follows:

```text
# to create a set of configs and installs under ``docs/source/example-code/build/nodes`` run
./gradlew docs/source/example-code:deployNodes
# to open up two new terminals with the two nodes run
./docs/source/example-code/build/nodes/runnodes
# followed by the same commands as before:
./docs/source/example-code/build/install/docs/source/example-code/bin/client-rpc-tutorial Print
./docs/source/example-code/build/install/docs/source/example-code/bin/client-rpc-tutorial Visualise
```
With regards to the start flow RPCs, there is an extra layer of security whereby the flow to be executed has to be
                annotated with `@StartableByRPC`. Flows without this annotation cannot execute using RPC.

See more on security in secure-coding-guidelines, node configuration in corda-configuration-file and
                Cordformation in running-a-node.

