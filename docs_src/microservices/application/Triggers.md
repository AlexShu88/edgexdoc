Triggers determine how the app functions pipeline begins execution. The trigger is determined by the `configuration.toml` file located in the `/res` directory under a section called `[Binding]`. Check out the [Configuration Section](../GeneralAppServiceConfig/) for more information about the toml file.

## EdgeX Message Bus Trigger

An EdgeX Message Bus trigger will execute the pipeline every time data is received from the configured Edgex Message Bus `SubscribeTopic`.  The EdgeX Message Bus is the central message bus internal to EdgeX and has a specific message envelope that wraps all data published to this message bus.

There currently are three implementations of the EdgeX Message Bus available to be used. These are `ZeroMQ`, `MQTT` & `Redis Streams`. The implementation type is selected via the `[MessageBus]` configuration described below.

### Type and Topic configuration 
Here's an example:

```toml
[Binding]
Type="edgex-messagebus" # also can use legacy messagebus for type
SubscribeTopic="events"
PublishTopic=""
```
The `Type=` is set to `edgex-messagebus` trigger type or the legacy type of `messagebus`. EdgeX Core Data is publishing data to the `events` topic. So to receive data from core data, you can set your `SubscribeTopic=` either to `""` or `"events"`. You may also designate a `PublishTopic=` if you wish to publish data back to the message bus. `edgexcontext.Complete([]byte outputData)` - Will send data back to the message bus on the topic specified by the `PublishTopic=` property

### Message bus connection configuration
The other piece of configuration required are the connection settings:
```toml
[MessageBus]
Type = 'zero' # message bus type (i.e zero for ZMQ, mqtt for MQTT or redisstreams for Redis Streams)
    [MessageBus.PublishHost]
        Host = '*'
        Port = 5564
        Protocol = 'tcp'
    [MessageBus.SubscribeHost]
        Host = 'localhost'
        Port = 5563
        Protocol = 'tcp'
```
By default, `EdgeX Core Data` publishes data to the `events`  topic using ZMQ on port 5563. The publish host is used if publishing data back to the message bus. As stated above there are three implementations you can choose from. These type values are as follows:

```
zero - for ZeroMQ
mqtt - for MQTT (Requires a MQTT Broker running and Core-Data configure to use MQTT)
redisstreams - for Redis Streams (Requires Redis running and Core-Data configure to use Redis Streams)
```

!!! important
    When using ZMQ for the message bus, the Publish Host **MUST** be different for every topic you wish to publish to since the SDK will bind to the specific port. 5563 for example cannot be used to publish since `EdgeX Core Data` has bound to that port. Similarly, you cannot have two separate instances of the app functions SDK running publishing to the same port.

!!! note
    When using MQTT for the message bus, there is additional configuration required for specifying the MQTT specific options. 

Here is example `MessageBus` configuration when using MQTT as the message bus:

```toml
[MessageBus]
Type = "mqtt"
    [MessageBus.SubscribeHost]
        Host = "localhost"
        Port = 1883
        Protocol = "tcp"
    [MessageBus.PublishHost]
        Host = "localhost"
        Port = 1883
        Protocol = "tcp"
    [MessageBus.Optional]
        # MQTT Specific options
        Username =""
        Password =""
        ClientId ="AppService"
        # Connection information
        Qos          =  "0" # Quality of Sevice 0 (At most once), 1 (At least once) or 2 (Exactly once)
        KeepAlive    =  "10" # Seconds (must be 2 or greater)
        Retained     = "false"
        AutoReconnect  = "true"
        ConnectTimeout = "5" # Seconds
        SkipCertVerify = "false"
        CertFile       = ""
        KeyFile        = ""
        KeyPEMBlock    = ""
        CertPEMBlock   = ""
```

!!! note
    The MQTT `MessageBus` implementation doesn't yet support retrieving secrets from the `Secret Store`. Thus the secrets for secure connection are currently in plain text in the configuration file.

## MQTT Trigger

A MQTT trigger will execute the pipeline every time data is received from the configured external MQTT broker on the configured subscribe topic.  

!!! note
    The data received from the external MQTT broker is not wrapped with any metadata known to EdgeX. The data is handled as JSON or CBOR. The data is assumed to be JSON unless the first byte in the data is not a `{` , in which case it is then assumed to be CBOR.

!!! note
    The data received, encoded as JSON or CBOR, must match the `TargetType` define by your application service. The default  `TargetType` is an `Edgex Event`. See [TargetType](../AdvancedTopics/#target-type) for more details.

### Type and Topic configuration 
Here's an example:
```toml
[Binding]
Type="external-mqtt" 
SubscribeTopic="some-request"
PublishTopic=""
```
The `Type=` is set to `external-mqtt`. To receive data from the external MQTT Broker you must set your `SubscribeTopic=` to the appropriate topic that the external publisher is using. You may also designate a `PublishTopic=` if you wish to publish data back to the external MQTT Broker. `edgexcontext.Complete([]byte outputData)` - Will send data back to back to the external MQTT Broker on the topic specified by the `PublishTopic=` property

### MQTT Broker configuration
The other piece of configuration required are the MQTT Broker connection settings:
```toml
[MqttBroker]
	Url = "tcp://localhost:1883" #  fully qualified URL to connect to the MQTT broker
	ClientId = "AppService" 
	ConnectTimeout = "5s" # 5 seconds
	AutoReconnect = true
	KeepAlive = 10 # Seconds (must be 2 or greater)
	QoS = 0 # Quality of Service 0 (At most once), 1 (At least once) or 2 (Exactly once)
	Retain = true
	SkipCertVerify = false
	SecretPath = "mqtt-trigger" 
	AuthMode = "none" # Options are "none", "cacert" , "usernamepassword", "clientcert".
```
## HTTP Trigger

Designating an HTTP trigger will allow the pipeline to be triggered by a RESTful `POST` call to `http://[host]:[port]/api/v1/trigger/`. 

`edgexcontext.Complete([]byte outputData)` - Will send the specified data as the response to the request that originally triggered the HTTP Request. 

!!! note
    The HTTP trigger uses the `content-type` from the HTTP Header to determine if the data is JSON or CBOR encoded and the optional `X-Correlation-ID` to set the correlation ID for the request.

!!! note
    The data received, encoded as JSON or CBOR, must match the `TargetType` defined by your application service. The default  `TargetType` is an `Edgex Event`. See [TargetType](../AdvancedTopics/#target-type) for more details.

## Custom Triggers

It is also possible to define your own trigger and register a factory function for it with the SDK.  You can then configure the trigger by registering a factory function to build it along with a name to use in the config file.  These triggers can be registered with:

```go
sdk.RegisterCustomTriggerFactory("my-trigger-name", myFactoryFunc) 
```

!!! note
    You can NOT override trigger names built into the SDK ("http", "edgex-messagebus", "external-mqtt") for a custom trigger.

The trigger factory function is bound to an instance of a trigger configuration struct that is provided by the SDK:

```go
type TriggerConfig struct {
	Config           *common.ConfigurationStruct
	Logger           logger.LoggingClient
	ContextBuilder   TriggerContextBuilder
	MessageProcessor TriggerMessageProcessor
}
```

This type carries a pointer to the internal edgex configuration and logger, along with two functions:

- `ContextBuilder` builds an `*appcontext.Context` from a message envelope you construct.
- `MessageProcessor` exposes a function that sends your message envelope and context built above into the edgex function pipeline.

The custom trigger constructed here will then need to implement the trigger interface so that the SDK can invoke it:

```go
type Trigger interface {
	Initialize(wg *sync.WaitGroup, ctx context.Context, background <-chan types.MessageEnvelope) (bootstrap.Deferred, error)
}
```

This leaves a lot of flexibility for how you want the trigger to behave (for example you could write a trigger to watch for file changes, or run on a timer).  Below is a sample implementation of a trigger to read lines from os.Stdin and pass the captured string through the edgex function pipeline.  In this case the target type for the service is `&[]byte{}`.

```go
type stdinTrigger struct{
	tc appsdk.TriggerConfig
}

func (t *stdinTrigger) Initialize(wg *sync.WaitGroup, ctx context.Context, background <-chan types.MessageEnvelope) (bootstrap.Deferred, error) {
    msgs := make(chan []byte)

    ctx, cancel := context.WithCancel(context.Background())

    receiveMessage := true

    go func() {
        fmt.Print("> ")
        rdr := bufio.NewReader(os.Stdin)
        for receiveMessage {
            s, err := rdr.ReadString('\n')
            s = strings.TrimRight(s, "\n")

            if err != nil {
                t.tc.Logger.Error(err.Error())
                continue
            }

            msgs <- []byte(s)
        }
    }()

    go func() {
        for receiveMessage {
            select {
            case <-ctx.Done():
                receiveMessage = false

            case m := <-msgs:
                go func() {
                    env := types.MessageEnvelope{
                        Payload: m,
                    }

                    ctx := t.tc.ContextBuilder(env)

                    err := t.tc.MessageProcessor(ctx, env)

                    if err != nil {
                        t.tc.Logger.Error(err.Error())
                    }
                }()
            }
        }
    }()

    return func() {
        cancel()
    }, nil
}
```

This trigger can then be registered by calling:

```go
edgexSdk.RegisterCustomTriggerFactory("custom-stdin", func(config appsdk.TriggerConfig) (appsdk.Trigger, error) {
    return &stdinTrigger{
        tc: config,
    }, nil
})
```
A complete working example can be found [here](https://github.com/edgexfoundry/edgex-examples/tree/master/application-services/custom/custom-trigger)