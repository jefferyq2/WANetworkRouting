# WANetworkRouting

[![Version](https://img.shields.io/cocoapods/v/WANetworkRouting.svg?style=flat)](http://cocoapods.org/pods/WANetworkRouting)
[![License](https://img.shields.io/cocoapods/l/WANetworkRouting.svg?style=flat)](http://cocoapods.org/pods/WANetworkRouting)
[![Platform](https://img.shields.io/cocoapods/p/WANetworkRouting.svg?style=flat)](http://cocoapods.org/pods/WANetworkRouting)

**Developed and Maintained by [ipodishima](https://github.com/ipodishima) Founder & CTO at [Wasappli Inc](http://wasapp.li).**

**Sponsored by [Wisembly](http://wisembly.com/en/)**

A routing library to fetch objects from an API and map them to your app

- [x] Highly customizable: network request, authentication and mapping layers are separate, you can create your own.
- [x] Default network request layer built on top of AFNetworking 3.0
- [x] Default mapping layer built on top of WAMapping
- [x] Built-in object router
- [x] Different configurations available
- [x] Tested

Go visit the [wiki](https://github.com/wasappli/WANetworkRouting/wiki) for more details about `WANetworkRouting` advanced use.

`WANetworkRouting` is a library which turns `GET enterprises/:itemID` to `Enterprise : NSManagedObject` from a simple configuration step.

## Install and use
### Cocoapods
Use Cocoapods, this is the easiest way to install the router.

`pod 'WANetworkRouting'`

`#import <WANetworkRouting/WANetworkRouting.h>` 

### `WANetworkRoutingManager`
`WANetworkRoutingManager` is the core component of `WANetworkRouting`. It handles all the requests for you.

The initialization should contains at least a request manager. `WANetworkRouting` has a built in manager `WAAFNetworkingRequestManager` built on top of `AFNetworking 3.0`.

```objc
// Create a request manager
WAAFNetworkingRequestManager *requestManager = [WAAFNetworkingRequestManager new];

// Create the network routing manager 
WANetworkRoutingManager *routingManager = [WANetworkRoutingManager managerWithBaseURL:[NSURL URLWithString:@"http://baseURL.com"]
                                                                       requestManager:requestManager
                                                                       mappingManager:nil
                                                                authenticationManager:nil];

```

** Note ** This `routingManager` will have not mapping layer nor authentication layer. This example demonstrates how easy it is to test your API before configuring the mapping.

You can then perform any operations among `GET|POST|PUT|PATCH|DELETE|HEAD`. For eg:

- Fetch all the enterprises

```objc
[routingManager getObjectsAtPath:@"enterprises"
                      parameters:nil
                         success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                             NSDictionary *json = response.responseObject;
                             // Do something with the JSON
                         }
                         failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                         }];
```

- Update an enterprise

```objc
[routingManager putObject:nil
                     path:@"enterprises/1"
               parameters:@{@"key": @"newValue"}
                  success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                  }
                  failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                  }];
```

### Add mapping layer
This is where things become to be interesting

The idea is to add a mapping layer so that your json gets mapped as objects. This is done by using a default mapping service built on top of [`WAMapping`](https://github.com/wasappli/WAMapping). I strongly encourage you to read the docs about how to configure the mapping.

There are 3 steps:

- Configure the mapping,
- Configure the response descriptor,
- Optional: configure the request descriptor.

#### Step 1: configure the mapping

Have a look at [`WAMapping`](https://github.com/wasappli/WAMapping) for every details about the mapping:

```objc
// Specify a store to use between WAMemoryStore, WANSCodingStore, WACoreDataStore, your own store.
WAMemoryStore *memoryStore = [[WAMemoryStore alloc] init];

// Create both a mapper and a reverse mapper
WAMapper *mapper               = [WAMapper newMapperWithStore:memoryStore];
WAReverseMapper *reverseMapper = [WAReverseMapper new];

// Add a default date formatter on mapper
id(^toDateMappingBlock)(id ) = ^id(id value) {
    if ([value isKindOfClass:[NSString class]]) {
        return [dateFormatter dateFromString:value];
    }

    return value;
};

[mapper addDefaultMappingBlock:toDateMappingBlock
           forDestinationClass:[NSDate class]];
           
// Create the mapping description for `Enterprise`
WAEntityMapping *enterpriseMapping = [WAEntityMapping mappingForEntityName:@"Enterprise"];
enterpriseMapping.identificationAttribute = @"itemID";
[enterpriseMapping addAttributeMappingsFromDictionary:@{
                                                        @"id": @"itemID",
                                                        @"name": @"name",
                                                        @"address.street_number": @"streetNumber"}];
// Create the mapping manager
WAMappingManager *mappingManager = [WAMappingManager mappingManagerWithMapper:mapper
                                                                reverseMapper:reverseMapper];
                                                                
WAAFNetworkingRequestManager *requestManager = [WAAFNetworkingRequestManager new];

// Create the network routing manager 
WANetworkRoutingManager *routingManager = [WANetworkRoutingManager managerWithBaseURL:[NSURL URLWithString:@"http://baseURL.com"]
                                                                       requestManager:requestManager
                                                                       mappingManager:mappingManager
                                                                authenticationManager:nil];
```

#### Step 2: configure the response descriptors
This will map a path with a mapping. For example, when fetching `/enterprises`, you would get a JSON as

```
{
	"array_of_enterprises": [{
		"name": "Enterprise 1"
	}, {
		"name": "Enterprise 2"
	}]
}
```

The request descriptor would then be as follows:

```objc
WAResponseDescriptor *enterprisesResponseDescriptor =
[WAResponseDescriptor responseDescriptorWithMapping:enterpriseMapping
                                             method:WAObjectRequestMethodGET
                                        pathPattern:@"enterprises" // The path on the URL
                                            keyPath:@"array_of_enterprises"]; // The key path to access the enterprises on JSON
```

For a `GET` or a `PUT` on an enterprise, which would be returned as

```
{
	"name": "Enterprise 1"
}
```

```objc
WAResponseDescriptor *singleEnterpriseResponseDescriptor =
[WAResponseDescriptor responseDescriptorWithMapping:enterpriseMapping
                                             method:WAObjectRequestMethodGET | WAObjectRequestMethodPUT // Specify multiple methods
                                        pathPattern:@"enterprises/:itemID" // The path
                                            keyPath:nil]; // The response would directly returns the object
```

Please note the syntax `enterprises/:itemID`: `:` means that the value would be replaced dynamically knowning that `itemID` is the property name on client side.

Finally, add the response descriptors to the mapping manager

```objc
[mappingManager addResponseDescriptor:enterprisesResponseDescriptor];
[mappingManager addResponseDescriptor:singleEnterpriseResponseDescriptor];
```

You can now use `WANetworkRouting` as this:

```objc
[routingManager getObjectsAtPath:@"enterprises"
                      parameters:nil
                         success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                             NSArray<Enterprise *> *enterprises = mappedObjects;
                             // Do something with the enterprises
                         }
                         failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                         }];
```

#### Step 3: configure the request descriptor
This step is optional, but allows you to reverse map an object to send it to the server.
Rather than creating by yourself the parameters dictionary on a `POST` for example, you could write:

```objc
WARequestDescriptor *enterpriseRequestDescriptor =
[WARequestDescriptor requestDescriptorWithMethod:WAObjectRequestMethodPOST
                                     pathPattern:@"enterprises" // The path on the URL
                                         mapping:enterpriseMapping
                                  shouldMapBlock:nil // On optional block which let you configure rather you want to reverse map a relation ship or not
                                  requestKeyPath:nil]; // The key path for the final dictionary
                                  
[mappingManager addRequestDescriptor:enterpriseRequestDescriptor];
```

Please note that the `requestKeyPath` fits with how your object would be wrapped on the parameters. If nil, the dictionary is the object as dictionary.

```
{
	"requestKeyPathValue": {
		"name": "Enterprise 1"
	}
}
```

This allows you to post an object like this

```objc
Enterprise *enterprise = [Enterprise new];
enterprise.name = @"Test";
        
[routingManager postObject:enterprise
                      path:@"enterprises"
                parameters:nil
                   success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                   }
                   failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                   }];
```

This will reverse map the `enterprise` to a dictionary and send it as parameters on `POST enterprise`. All magically without anymore work!

**Note**: If `enterprise` has no value for its `identificationAttribute` (see the mapping), the enterprise object would be automatically deleted from the store since the object returned by the server would be a new one with an `identificationAttribute`. Said in other words: you do not have to deal with potential duplicates.

### Add routing layer

We just saw this call

```objc
[routingManager postObject:enterprise
                      path:@"enterprises"
                parameters:nil
                   success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                   }
                   failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                   }];
```

Wouldn't it be great to write instead?

```objc
[routingManager postObject:enterprise
                      path:nil // No value for path
                parameters:nil
                   success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                   }
                   failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, id<WANRErrorProtocol> error) {
                   }];
```

Here comes the router!

```objc
WANetworkRoute *postEnterpriseRoute =
[WANetworkRoute routeWithObjectClass:[Enterprise class]
                         pathPattern:@"enterprises"
                              method:WAObjectRequestMethodPOST];

[routingManager.router postEnterpriseRoute];
```

Each time you are trying to `POST` an object of class `Enterprise`, it will supply the `enterprises` path for you.

Here is an other example with `GET` and `PUT` routes:

```objc
WANetworkRoute *enterpriseRoute =
[WANetworkRoute routeWithObjectClass:[Enterprise class]
                         pathPattern:@"enterprises/:itemID"
                              method:WAObjectRequestMethodGET | WAObjectRequestMethodPUT];

[routingManager.router addRoute:enterpriseRoute];
```

### Add authentication layer

This layer is important if your API has some authentication. Basically you should have a login/signup endpoint which returns some kind of token that you can renew if the requests gets an error like `401: token expired`.
By implementing the simple `WARequestAuthenticationManagerProtocol` protocol, and passing an instance of your class to the router manager, it will:

- Ask to authenticate the `NSMutableURLRequest`. Should be a token on `Authorization` HTTP header field.
- Ask if a request should be replayed: you received `401: token expired` for example then yes.
- Ask you to authenticate (renew the authorization somehow) and replay the request (`[routingManager enqueueRequest:]`) (The request will automatically be re authorized for you).

This will allow the routing manager to run every requests without any surprise when authentication has expired!

## Errors

Each api has a way to describe errors details others than the http codes (404, 403, 401, ...). For example, you could have a response like:

```
{
	"error": {
		"error_code": 4,
		"error_description": "The token is expired"
	}
}
```

The `WANetworkRouting` allows you to customize the error handling and retrieve both code and error description. Let's see how:
It's an other protocol (again): `WANRErrorProtocol`

### Create your own class
The routing manager comes with a default error class which does nothing except being allocated. 

```objc
@interface MyAPIError : NSObject <WANRErrorProtocol>
@end

@implementation MyAPIError 

- (instancetype)initWithOriginalError:(NSError *)error response:(WAObjectResponse *)response {
    self = [super init];
    
    if (self) {
        self->_originalError = error;
        self->_response      = response;
        self->_finalError    = error;
        
        NSDictionary *errorDescription = response.responseObject[@"error"];
        NSInteger errorCode = [errorDescription[@"error_code"] integerValue];
        NSString *errorDesc = errorDescription[@"error_description"];
        
        self->_finalError = [NSError errorWithDomain:MyDomain
                                                code:errorCode
                                            userInfo:@{
                                                       NSLocalizedDescriptionKey: errorDesc
                                                       }];
    }
    
    return self;
}

@end

```

### Register the class

```objc
WAAFNetworkingRequestManager *requestManager = [WAAFNetworkingRequestManager new];
requestManager.errorClass = [MyAPIError class];
```

### Use the class

```objc
[routingManager postObject:enterprise
                      path:nil
                parameters:nil
                   success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                   }
                   failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, MyAPIError *error) {
                   		if (error.finalError.errorCode == 4) {
                   			// Token has expired :/
                   		}
                   }];
```

## Delete a ressource

A quick word about deletion:

If you write
```objc
[routingManager deleteObject:enterprise
                        path:nil
                  parameters:nil
                     success:^(WAObjectRequest *objectRequest, WAObjectResponse *response, NSArray *mappedObjects) {
                   }
                     failure:^(WAObjectRequest *objectRequest, WAObjectResponse *response, MyAPIError *error) {
                   }];
```

This will automatically delete enterprise from the store on success! Be careful that if you delete an object only from it's ressource, it's up to you to delete it from your store (`[routingManager deleteObject:nil path:@"enterprises/1"...]`).

## Get the HTTP code and header fields from the response

```objc
// WAObjectResponse *response

NSInteger responseStatusCode = response.urlResponse.statusCode;
NSDictionary *httpHeaderFields = response.urlResponse.httpHeaderFields;
```

## Inspiration
Let's be honest, you'll find some things which looks like [Restkit](https://github.com/RestKit/RestKit).
I used RestKit for a while on a very big projects, but it tends to be too much magic and hard to maintain for our need which remains simple.
For example: upgrading `AFNetworking` on RestKit is just... Well, something we wait for months!

By compartimenting the layers, I hope to fix this issue!

#Contributing : Problems, Suggestions, Pull Requests?

Please open a new Issue [here](https://github.com/Wasappli/WANetworkRouting/issues) if you run into a problem specific to WANetworkRouting.

For new features pull requests are encouraged and greatly appreciated! Please try to maintain consistency with the existing code style. If you're considering taking on significant changes or additions to the project, please ask me before by opening a new issue to have a chance for a merge.
Please also run the tests before ;)

#That's all folks !

- If your are happy don't hesitate to send me a tweet [@ipodishima](http://twitter.com/ipodishima)!
- Distributed under MIT licence.
- Follow Wasappli on [facebook](https://www.facebook.com/wasappli)