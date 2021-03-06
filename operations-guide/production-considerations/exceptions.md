# Exceptions

Exception handling is a well known concept when developing software.
Dealing with exceptions in a distributed application landscape is however a little more challenging then we are typically used to.
Especially when it comes to failures when handling a command or a query, messages that are bound to have a return value,
 we should be conscious about how we throw exceptions.
 
## Handler Execution Exception

The `HandlerExceutionException` marks an exception which originates from a message handling member.
Since an [Event](../../configuring-infrastructure-components/messaging-concepts/messaging-concepts.md#events)
 message is unidirectional, handling an event does not include any return values.
As such,
 the `HandlerExceutionException` should only be returned as an exceptional result from handling a command or a query.
Axon provides a more concrete implementation of this exception for failed command and query handling,
 respectively the `CommandExecutionException` and `QueryExecutionException`. 

The usefulness of a dedicated handler execution exception becomes clearer in a distributed application environment were,
 for example, 
 there is a dedicated application for dealing with commands and another application tasked with the query side.
Due to the application segregation, you loose any certainty that both applications can access the same classes,
 which thus also hold for any exception classes.
To support and encourage this decoupling,
 Axon will generify any exception which is a result of Command or Query handling.
 
To maintain support for conditional logic dependent on the type of exception thrown in a distributed scenario,
 it is possible to provide details in a `HandlerExceutionException`.
It is thus recommended to throw a `CommandExecutionException`/`QueryExecutionException` with the required details,
 when command/query handling fails.
This behaviour could be supported generically by implementing
 [interceptors](../../configuring-infrastructure-components/messaging-concepts/message-intercepting.md) which perform 
 exception wrapping for you.
