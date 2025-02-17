<p align="center">
  <h3 align="center">CodableWrapper</h3>
  <p align="center">
    Codable + PropertyWrapper = @Codec("encoder", "decoder") var cool: Bool = true
  </p>
</p>
<ol>
  <li><a href="#about-the-project">About</a></li>
  <li><a href="#feature">Feature</a></li>
  <li><a href="#installation">Installation</a></li>
  <li><a href="#example">Example</a></li>
  <li><a href="#how-it-works">How it works</a></li>
  <li>
    <a href="#usage">Usage</a>
    <ul>
      <li><a href="#defaultvalue">DefaultValue</a></li>
      <li><a href="#codingkeys">CodingKeys</a></li>
      <li><a href="#basictypebridge">BasicTypeBridge</a></li>
      <li><a href="#transformer">Transformer</a></li>
    </ul>
  </li>
</ol>

## [中文说明](./README-zhCN.md)

# Swift 5.9 Macro Support

Is **<mark>In development</mark>**, chek [swift5.9-macro branch](https://github.com/winddpan/CodableWrapper/tree/swift5.9-macro)

## About

* This project is use `PropertyWrapper` to improve your `Codable` use experience.
* Simply based on `JSONEncoder` `JSONDecoder`.
* Powerful and simplifily API than  [BetterCodable](https://github.com/marksands/BetterCodable) or [CodableWrappers](https://github.com/GottaGetSwifty/CodableWrappers).

## Feature

* Default value supported
* Basic type convertible, between `String`  `Bool` `Number` 
* Custom key support
* Fix parsing failure due to missing fields from server
* Fix parsing failure due to mismatch Enum raw value
* Custom transform

## Installation

#### Cocoapods

``` pod 'CodableWrapper' ```

#### Swift Package Manager

``` https://github.com/winddpan/CodableWrapper ```

## Example

```Swift
enum Animal: String, Codable {
    case dog
    case cat
    case fish
}

struct ExampleModel: Codable {
    @Codec("aString")
    var stringVal: String = "scyano"

    @Codec("aInt")
    var intVal: Int = 123456

    @Codec var defaultArray: [Double] = [1.998, 2.998, 3.998]

    @Codec var bool: Bool = false

    @Codec var unImpl: String?

    @Codec var animal: Animal = .dog
}

let json = #"{"aString": "pan", "aInt": "233", "bool": "1", "animal": "cat"}"#

let model = try JSONDecoder().decode(ExampleModel.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.stringVal, "pan")
XCTAssertEqual(model.intVal, 233)
XCTAssertEqual(model.defaultArray, [1.998, 2.998, 3.998])
XCTAssertEqual(model.bool, true)
XCTAssertEqual(model.unImpl, nil)
XCTAssertEqual(model.animal, .cat)
```

*For more examples, please check the unit tests or Playground*

## How it works

```Swift
struct DataModel: Codable {
    @Codec var stringVal: String = "OK"
}

/* pseudocode from Swift open source lib: Codable.Swift -> */
struct DataModel: Codable {
    private var _stringVal = Codec<String>(defaultValue: "OK")

    var stringVal: String {
        get {
            return _stringVal.wrappedValue
        }
        set {
            _stringVal.wrappedValue = newValue
        }
    }

    enum CodingKeys: CodingKey {
        case stringVal
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        /* decode `newStringVal` */
        /* remember `newStringVal`: Thread.current.lastCodableWrapper = wrapper */
        /*
         extension KeyedDecodingContainer {
            func decode<Value>(_ type: Codec<Value>.Type, forKey key: Key) throws -> Codec<Value> {
                ...
                let wrapper = Codec<Value>(unsafed: ())
                Thread.current.lastCodableWrapper = wrapper
                ...
            }
         }
         */
        let newStringVal = try container.decode(Codec<String>.self, forKey: CodingKeys.stringVal)

        /* old `_stringVal` deinit */
        /* old `_stringVal` invokeAfterInjection called: transform old `_stringVal` Configs to `newStringVal` */
        /* 
         deinit {
             if !unsafeCreated, let construct = construct, let lastWrapper = Thread.current.lastCodableWrapper as? Codec<Value> {
                 lastWrapper.invokeAfterInjection(with: construct)
                 Thread.current.lastCodableWrapper = nil
             }
         }
        */
        self._stringVal = newStringVal
    }
}
```

## Usage

#### DefaultValue

> DefaultValue should implement `Codable` protocol

```swift
struct ExampleModel: Codable {
    @Codec var bool: Bool = false
}

let json = #"{"bool":"wrong value"}"#

let model = try JSONDecoder().decode(ExampleModel.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.bool, false)
```

#### Auto snake camel convert

```swift
struct ExampleModel: Codable {
    @Codec var snake_string: String = ""
    @Codec var camelString: String = ""
}

let json = #"{"snakeString":"snake", "camel_string": "camel"}"#

let model = try JSONDecoder().decode(ExampleModel.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.snake_string, "snake")
XCTAssertEqual(model.camelString, "camel")
```

#### CodingKeys

> Decoding:  try each CodingKey until succeed
> Encoding:  use first CodingKey as Dictionary key

```swift
struct ExampleModel: Codable {
    @Codec("int_Val", "intVal")
    var intVal: Int = 123456

    @Codec("intOptional", "int_optional")
    var intOptional: Int?
}

let json = #"{"int_Val": "233", "int_optional": 234}"#

let model = try JSONDecoder().decode(ExampleModel.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.intVal, 233)
XCTAssertEqual(model.intOptional, 234)

let data = try JSONEncoder().encode(model)
let jsonObject = try JSONSerialization.jsonObject(with: data, options: []) as! [String: Any]
XCTAssertEqual(jsonObject["int_Val"] as? Int, 233)
XCTAssertEqual(jsonObject["intOptional"] as? Int, 234)
```

#### Basic type bridging

```swift
struct ExampleModel: Codable {
    @Codec var int: Int?
    @Codec var string: String?
    @Codec var bool: Bool?
}

let json = #"{"int": "1", "string": 2, "bool": "true"}"#

let model = try JSONDecoder().decode(ExampleModel.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.int, 1)
XCTAssertEqual(model.string, "2")
XCTAssertEqual(model.bool, true)
```

#### Transformer

```swift
struct User: Codable {
    @Codec(transformer: SecondDateTransform())
    var registerDate: Date?
}       
let date = Date()
let json = #"{"sencondsDate": \(date.timeIntervalSince1970)}"#

let user = try JSONDecoder().decode(User.self, from: json.data(using: .utf8)!)
XCTAssertEqual(model.sencondsDate?.timeIntervalSince1970, date.timeIntervalSince1970)
```

> It also support custom transformer, your `CustomTransformer` only need to comfirm to `TransfromType`

## License

Distributed under the MIT License. See `LICENSE` for more information.
