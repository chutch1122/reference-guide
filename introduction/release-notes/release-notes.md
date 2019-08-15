# Release Notes

All the enhancements and features which have been introduced to our major and minor release are documented here.
This covers improvements to [Axon Framework](https://github.com/AxonFramework/AxonFramework),
 [Axon Server](https://axoniq.io/product-overview/axon-server) 
 and the [Axon Framework Extensions](https://github.com/AxonFramework?utf8=%E2%9C%93&q=extensions&type=&language=),
 since all three follow the same release cadence. 

## Release 4.0

 * The package structure of Axon Framework has changed drastically with the aim to provide users the option to pick and choose.
   For example, if only the messaging components of framework are required, one can directly depend on the `axon-messaging` package.
 * In part with the package restructure, all components which leverage another framework to provide something extra have been given their own repository.
   These repositories are called the [Axon Framework Extensions](https://github.com/AxonFramework?utf8=%E2%9C%93&q=extensions&type=&language=).
 * The configuration of Event Processor has been replaced and greatly fine tuned with the addition of the `EventProcessingConfigurer`.     
 * Some new defaults have been introduced in release 4.0, like a bias towards expecting a connection with Axon Server.
   Another important chance is the switch from defaulting to Tracking Processors instead of Subscribing Processors.
 * The notion of a `CommandResultMessage` has been introduced as a dedicated message towards the result of command handling.
 * To simplify configuration and more easily overcome deprecation,
    the [Builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) has been implemented for all infrastructure components.
 
For more details, check the list of issues [here](https://github.com/AxonFramework/AxonFramework/milestone/28?closed=1).