# ObjSer: object graph serialisation

ObjSer is a binary data-interchange format. It differs from formats such as [JSON](http://json.org) and [MessagePack](http://msgpack.org) in that it is able to represent *arbitrary* object graphs, including those containing cycles or multiple instances of the same object, without duplication.

Read the [specification](spec.md).

## Implementations

- [**Swift**](https://github.com/greg/objser-swift) (reference implementation, compliant but feature-incomplete)

