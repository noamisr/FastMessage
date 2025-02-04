# Using FastMessage

FastMessage is an extension package for [MessageFlux](https://messageflux.readthedocs.io).
It is an easy way to create a ```PipelineHandlerBase``` and put it in a ```PipelineService```

## Limitations

With ```FastMessage``` you can register only a single callback for each input device, 
meaning the service will expect a single type (schema) of messages for each input device.

Multiple callbacks can send messages to the same output device.

Also - Positional Only Arguments are not allowed on the callbacks

## Examples

### Creating a FastMessage

```python
from pydantic import BaseModel

from fastmessage import FastMessage

fm = FastMessage()


@fm.map()
def do_something(x: int, y: str):
    pass  # do something with x and y


class SomeModel(BaseModel):
    x: int
    y: str


@fm.map(output_device='some_output_queue')
def do_something_else(m: SomeModel, a: int):
    return "some_value"  # do somthing with m and a

```

### Creating the service runner

```python
from messageflux.iodevices.rabbitmq.rabbitmq_input_device import RabbitMQInputDeviceManager
from messageflux.iodevices.rabbitmq.rabbitmq_output_device import RabbitMQOutputDeviceManager

from fastmessage import FastMessage

fm = FastMessage()

# Code for creating FastMessage here...


input_device_manager = RabbitMQInputDeviceManager(hosts='my.rabbit.host',
                                                  user='username',
                                                  password='password')

output_device_manager = RabbitMQOutputDeviceManager(hosts='my.rabbit.host',
                                                    user='username',
                                                    password='password')

service = fm.create_service(input_device_manager=input_device_manager,
                            output_device_manager=output_device_manager)

service.start()
```

### Root Model
You can use a callback with a single param named ```__root__``` in order to mark it as the root type for the message.
in that case, the message will not expect the param name, and instead will expect the root model as type

```python
from pydantic import BaseModel

from fastmessage import FastMessage

fm = FastMessage()


class SomeModel(BaseModel):
    x: int
    y: str


@fm.map(input_device='some_other_queue')
def process_somemodel(__root__: SomeModel):
    pass # this method will expect messages with the SomeModel schema ({"x":1, "y":"some string"})  

```

## Special Param Types

There are three special types which you can annotate the arguments for the callback with.

* ```InputDeviceName``` - arguments annotated with this type will receive the name of the input device the message came from. useful for registering the same callback for several input devices
* ```Message``` - arguments annotated with this type will receive the raw message which came from the device
* ```MessageBundle``` - arguments annotated with this type will receive the complete MessageBundle (with device headers)

Notice that arguments annotated with these types MUST NOT have default values (Since they always have values).

### Example

```python
from pydantic import BaseModel

from fastmessage import FastMessage,InputDeviceName
from messageflux.iodevices.base import Message, MessageBundle
fm = FastMessage()


@fm.map(input_device='some_queue')
def do_something(i: InputDeviceName, m: Message, mb:MessageBundle, x:int):
    # i will be 'some_queue'
    # m will be the message that arrived
    # mb will be the MessageBundle that arrived
    # x will be the serialized value of the message
    pass  # do something
```

## Returning Multiple Results

You can make the function return multiple results, where each one is serialized as its own message to the output queue.

All you have to do, is return a ```MultipleReturnValues``` (which is a ```List```), and each item will be serialized as its own output message

### Example

```python
from fastmessage import FastMessage, MultipleReturnValues

fm = FastMessage()


@fm.map(input_device='some_queue', output_device='some_other_queue')
def do_something(x: int):
    return MultipleReturnValues([1, 'b', 3])  # will create 3 output messages, one for each item
```


## Returning Result to a different output device

You can make the function return a result to a different output device then the one in the decorator

You do this by using the ```FastMessageOutput``` class, and giving it the value to send, and the output device name to send to

### Example

```python
from fastmessage import FastMessage, FastMessageOutput

fm = FastMessage()


@fm.map(input_device='some_queue', output_device='default_output_device')
def do_something(x: int):
    return FastMessageOutput(value=1, output_device='other_output_device') # this will send the value 1 to 'other_output_device' instead of the default
```

