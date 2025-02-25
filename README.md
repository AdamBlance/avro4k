# <img src="https://github.com/avro-kotlin/avro4k/raw/main/src/main/graphics/logo.png" height=160>
[Avro](https://avro.apache.org/) format for [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization). This library is a port of [sksamuel's](https://github.com/sksamuel) Scala Avro generator [avro4s](https://github.com/sksamuel/avro4s).

![build-main](https://github.com/avro-kotlin/avro4k/workflows/build-main/badge.svg)
[<img src="https://img.shields.io/maven-central/v/com.github.avro-kotlin.avro4k/avro4k-core.svg?label=latest%20release"/>](http://search.maven.org/#search%7Cga%7C1%7Cavro4k)


## Introduction

Avro4k is a Kotlin library that brings support for Avro to the Kotlin Serialization framework. This library supports reading and writing to/from binary and json streams as well as supporting Avro schema generation. 

- [Generate schemas](https://github.com/avro-kotlin/avro4k#schemas) from Kotlin data classes. This allows you to use data classes as the canonical source for the schemas and generate Avro schemas from _them_, rather than define schemas externally and then generate (or manually write) data classes to match.
- Marshall data classes to / from instances of Avro Generic Records. The basic _structure type_ in Avro is the _IndexedRecord_ or it's more common subclass, the _GenericRecord_. This library will marshall data to and from Avro records. This can be useful for interop with frameworks like Kafka which provide serializers which work at the record level.
- [Read / Write](https://github.com/avro-kotlin/avro4k#input--output) data classes to input or output streams. Avro records can be serialized as binary (with or without embedded schema) or json, and this library provides _AvroInputStream_ and _AvroOutputStream_ classes to support data classes directly.
- [Support logical types](https://github.com/avro-kotlin/avro4k#types). This library provides support for the Avro [logical types](https://avro.apache.org/docs/1.11.0/spec.html#Logical+Types) out of the box, in addition to the _standard_ types supported by the Kotlin serialization framework.
- Add custom serializer for other types. With Avro4k you can easily add your own _AvroSerializer_ instances that provides schemas and serialization for types not supported by default.

## Schemas

Writing schemas manually through the Java based `SchemaBuilder` classes can be tedious for complex domain models. 
Avro4k allows us to generate schemas directly from data classes at compile time using the Kotlin Serialization library. 
This gives you both the convenience of generated code, without the annoyance of having to run a code generation step, as well as avoiding the performance penalty of runtime reflection based code.

Let's define some classes.

```kotlin
@Serializable
data class Ingredient(val name: String, val sugar: Double, val fat: Double)

@Serializable
data class Pizza(val name: String, val ingredients: List<Ingredient>, val vegetarian: Boolean, val kcals: Int)
```

To generate an Avro Schema, we need to use the `Avro` object, invoking `schema` and passing in the serializer generated by the Kotlin Serialization compiler plugin for your target class. This will return an `org.apache.avro.Schema` instance.

In other words:

```kotlin
val schema = Avro.default.schema(Pizza.serializer())
println(schema.toString(true))
```

Where the generated schema is as follows:

```json
{
   "type":"record",
   "name":"Pizza",
   "namespace":"com.github.avrokotlin.avro4k.example",
   "fields":[
      {
         "name":"name",
         "type":"string"
      },
      {
         "name":"ingredients",
         "type":{
            "type":"array",
            "items":{
               "type":"record",
               "name":"Ingredient",
               "fields":[
                  {
                     "name":"name",
                     "type":"string"
                  },
                  {
                     "name":"sugar",
                     "type":"double"
                  },
                  {
                     "name":"fat",
                     "type":"double"
                  }
               ]
            }
         }
      },
      {
         "name":"vegetarian",
         "type":"boolean"
      },
      {
         "name":"kcals",
         "type":"int"
      }
   ]
}
```
You can see that the schema generator handles nested data classes, lists, primitives, etc. For a full list of supported object types, see the table later.



### Overriding class name and namespace

Avro schemas for complex types (RECORDs) contain a name and a namespace. 
By default, these are the name of the class and the enclosing package name, but it is possible to customize these using the annotations `@AvroName` and `@AvroNamespace`.

For example, the following class:

```kotlin
package com.github.avrokotlin.avro4k.example

data class Foo(a: String)
```

Would normally have a schema like this:

```json
{
  "type":"record",
  "name":"Foo",
  "namespace":"com.github.avrokotlin.avro4k.example",
  "fields":[
    {
      "name":"a",
      "type":"string"
    }
  ]
}
```

However we can override the name and/or the namespace like this:

```kotlin
package com.github.avrokotlin.avro4k.example

@AvroName("Wibble")
@AvroNamespace("com.other")
data class Foo(val a: String)
```

And then the generated schema looks like this:

```json
{
  "type":"record",
  "name":"Wibble",
  "namespace":"com.other",
  "fields":[
    {
      "name":"a",
      "type":"string"
    }
  ]
}
```

Note: It is possible, but not necessary, to use both AvroName and AvroNamespace. You can just use either of them if you wish.





### Overriding a field name

The `@AvroName` annotation can also be used to override field names. 
This is useful when the record instances you are generating or reading need to have field names different from the Kotlin data classes. 
For example if you are reading data generated by another system or another language.

Given the following class.

```kotlin
package com.github.avrokotlin.avro4k.example

data class Foo(val a: String, @AvroName("z") val b : String)
```

Then the generated schema would look like this:

```json
{
  "type":"record",
  "name":"Foo",
  "namespace":"com.github.avrokotlin.avro4k.example",
  "fields":[
    {
      "name":"a",
      "type":"string"
    },
    {
      "name":"z",
      "type":"string"
    }    
  ]
}
```

Notice that the second field is z and not b.

Note: `@AvroName` does not add an alternative name for the field, but an override.
If you wish to have alternatives then you should use `@AvroAlias`.





### Adding properties and docs to a Schema

Avro allows a doc field, and arbitrary key/values to be added to generated schemas.
Avro4k supports this through the use of `@AvroDoc` and `@AvroProp` annotations.

These properties works on either complex or simple types - in other words, on both fields and classes. 

For example, the following code:

```kotlin
package com.github.avrokotlin.avro4k.example

@Serializable
@AvroDoc("hello, is it me you're looking for?")
data class Foo(@AvroDoc("I am a string")  val str: String, 
               @AvroDoc("I am a long")    val long: Long, 
                                          val int: Int)
```

Would result in the following schema:

```json
{  
  "type": "record",
  "name": "Foo",
  "namespace": "com.github.avrokotlin.avro4k.example",
  "doc":"hello, is it me you're looking for?",
  "fields": [  
    {  
      "name": "str",
      "type": "string",
      "doc" : "I am a string"
    },
    {  
      "name": "long",
      "type": "long",
      "doc" : "I am a long"
    },
    {  
      "name": "int",
      "type": "int"
    }
  ]
}
```

Similarly, for properties:

```kotlin
package com.github.avrokotlin.avro4k.example

@Serializable
@AvroProp("jack", "bruce")
data class Annotated(@AvroProp("richard", "ashcroft") val str: String, 
                     @AvroProp("kate", "bush")        val long: Long, 
                                                      val int: Int)
```

Would generate this schema:

```json
{
  "type": "record",
  "name": "Annotated",
  "namespace": "com.github.avrokotlin.avro4k.example",
  "fields": [
    {
      "name": "str",
      "type": "string",
      "richard": "ashcroft"
    },
    {
      "name": "long",
      "type": "long",
      "kate": "bush"
    },
    {
      "name": "int",
      "type": "int"
    }
  ],
  "jack": "bruce"
}
```

### Decimal scale and precision

In order to customize the scale and precision used by BigDecimal schema generators, 
you can add the `@ScalePrecision` annotation to instances of BigDecimal.

For example, this code:

```kotlin
@Serializable
data class Test(@ScalePrecision(1, 4) val decimal: BigDecimal)

val schema = Avro.default.schema(Test.serializer())
```

Would generate the following schema:

```json
{
  "type":"record",
  "name":"Test",
  "namespace":"com.foo",
  "fields":[{
    "name":"decimal",
    "type":{
      "type":"bytes",
      "logicalType":"decimal",
      "scale":"1",
      "precision":"4"
    }
  }]
}
```

### Avro Fixed

Avro supports the idea of fixed length byte arrays.
To use these we can either override the schema generated for a type to return Schema.Type.Fixed. 
This will work for types like String or UUID. 
You can also annotate a field with `@AvroFixed(size)`. 

For example:

```kotlin
package com.github.avrokotlin.avro4k.example

data class Foo(@AvroFixed(7) val mystring: String)

val schema = Avro.default.schema(Foo.serializer())
```

Will generate the following schema:

```json
{
  "type": "record",
  "name": "Foo",
  "namespace": "com.github.avrokotlin.avro4k.example",
  "fields": [
    {
      "name": "mystring",
      "type": {
        "type": "fixed",
        "name": "mystring",
        "size": 7
      }
    }
  ]
}
```

If you have a type that you always want to be represented as fixed, then rather than annotate every single location
 it is used, you can annotate the type itself.

```kotlin
package com.github.avrokotlin.avro4k.example

@AvroFixed(4)
@Serializable
data class FixedA(val bytes: ByteArray)

@Serializable
data class Foo(val a: FixedA)

val schema = Avro.default.schema(Foo.serializer())
```

And this would generate:

```json
{
  "type": "record",
  "name": "Foo",
  "namespace": "com.github.avrokotlin.avro4k.example",
  "fields": [
    {
      "name": "a",
      "type": {
        "type": "fixed",
        "name": "FixedA",
        "size": 4
      }
    }
  ]
}
```

### Transient Fields

The kotlinx.serialization framework does not support the standard @transient anotation to mark a field as ignored, but instead supports its own `@kotlinx.serialization.Transient` annotation to do the same job.
 Any field marked with this will be excluded from the generated schema.

For example, the following code:

```kotlin
package com.github.avrokotlin.avro4k.example

data class Foo(val a: String, @AvroTransient val b: String)
```

Would result in the following schema:

```json
{  
  "type": "record",
  "name": "Foo",
  "namespace": "com.github.avrokotlin.avro4k.example",
  "fields": [  
    {  
      "name": "a",
      "type": "string"
    }
  ]
}
```


## Types

Avro4k supports the Avro logical types out of the box as well as other common JDK types.

Avro has no understanding of Kotlin types, or anything outside of it's built in set of supported types, so all values must be converted to something that is compatible with Avro. 

For example a `java.sql.Timestamp` is usually encoded as a `Long`, and a `java.util.UUID` is encoded as a `String`.

Some values can be mapped in multiple ways depending on how the schema was generated. For example a `String`, which is usually encoded as an `org.apache.avro.util.Utf8` could also be encoded as an array of bytes if the generated schema for that field was `Schema.Type.BYTES`. 
Therefore some serializers will take into account the schema passed to them when choosing the avro compatible type.

The following table shows how types used in your code will be mapped / encoded in the generated Avro schemas and files. 
If a type can be mapped in multiple ways, it is listed more than once.

| JVM Type                   	   | Schema Type   	  | Logical Type     	 | Encoded Type             |
|--------------------------------|------------------|--------------------|--------------------------|
| String                       	 | STRING        	  | 	                  | Utf8                     |
| String                       	 | FIXED        	   | 	                  | GenericFixed             |
| String                       	 | BYTES        	   | 	                  | ByteBuffer               |
| Boolean                      	 | BOOLEAN       	  | 	                  | java.lang.Boolean        |
| Long                         	 | LONG          	  | 	                  | java.lang.Long           |
| Int                          	 | INT           	  | 	                  | java.lang.Integer        |
| Short                        	 | INT           	  | 	                  | java.lang.Integer        |
| Byte                         	 | INT           	  | 	                  | java.lang.Integer        |
| Double                       	 | DOUBLE        	  | 	                  | java.lang.Double         |
| Float                        	 | FLOAT         	  | 	                  | java.lang.Float          |
| UUID                         	 | STRING        	  | UUID             	 | Utf8                     |
| LocalDate                    	 | INT           	  | Date             	 | java.lang.Int            |
| LocalTime                    	 | INT           	  | Time-Millis      	 | java.lang.Int            |
| LocalDateTime                	 | LONG          	  | Timestamp-Millis 	 | java.lang.Long           |
| Instant                      	 | LONG          	  | Timestamp-Millis 	 | java.lang.Long           |
| Timestamp                    	 | LONG          	  | Timestamp-Millis 	 | java.lang.Long           |
| BigDecimal                   	 | BYTES         	  | Decimal<8,2>     	 | ByteBuffer               |
| BigDecimal                   	 | FIXED         	  | Decimal<8,2>     	 | GenericFixed             |
| BigDecimal                   	 | STRING         	 | Decimal<8,2>     	 | String                   |
| T? (nullable type)           	 | UNION<null,T> 	  | 	                  | null, T                  |
| ByteArray                  	   | BYTES         	  | 	                  | ByteBuffer               |
| ByteAray                  	    | FIXED         	  | 	                  | GenericFixed             |
| ByteBuffer                   	 | BYTES         	  | 	                  | ByteBuffer               |
| List[Byte]                   	 | BYTES         	  | 	                  | ByteBuffer               |
| Array<T>                     	 | ARRAY<T>      	  | 	                  | Array[T]                 |
| List<T>                      	 | ARRAY<T>      	  | 	                  | Array[T]                 |
| Set<T>                      	  | ARRAY<T>      	  | 	                  | Array[T]                 |
| Map[String, V]              	  | MAP<V>        	  | 	                  | java.util.Map[String, V] |
| data class T                 	 | RECORD        	  | 	                  | GenericRecord            |
| enum class             	       | ENUM          	  | 	                  | GenericEnumSymbol        |

In order to use logical types, annotate the value with an appropriate Serializer:

```kotlin
import com.github.avrokotlin.avro4k.Avro
import com.github.avrokotlin.avro4k.serializer.InstantSerializer
import kotlinx.serialization.Serializable
import java.time.Instant

@Serializable
data class WithInstant(
   @Serializable(with=InstantSerializer::class) val inst: Instant
)
```

All the logical type serializers are available in the package `com.github.avrokotlin.avro4k.serializer`. You can find additional serializers for [arrow](https://arrow-kt.io/) types in the
[avro4k-arrow](https://github.com/avro-kotlin/avro4k-arrow) project.

## Input / Output

### Formats

Avro supports four different encoding types [serializing records](https://avro.apache.org/docs/current/spec.html#Data+Serialization+and+Deserialization).

These are binary with schema, binary without schema, json and single object encoding.

In avro4k these are represented by an `AvroFormat` enum with three values - `AvroFormat.Binary` (binary no schema), `AvroFormat.Data` (binary with schema), and `AvroFormat.Json`.
The single object encoding is currently not supported. 

Binary encoding without the schema does not include field names, self-contained information about the types of individual bytes, nor field or record separators.
Therefore readers are wholly reliant on the schema used when the data was encoded but the format is by far the most compact. Binary encodings are [fast](https://www.slideshare.net/oom65/orc-files?next_slideshow=1).

Binary encoding with the schema is still quick to deserialize, but is obviously less compact. However, as the schema is included readers do not need to have access to the original schema.   

Json encoding is the largest and slowest, but the easist to work with outside of Avro, and of course is easy to view on the wire (if that is a concern).


### Serializing

Avro4k allows us to easily serialize data classes using an instance of `AvroOutputStream` which we write to, and close, just like you would any regular output stream. It is created by calling `openOutputStream` on an `Avro` instance.

When creating an output stream we specify the target, such as a File, Path, or another output stream. 
If nothing more is specified, the default encode format `AvroEncodeFormat.Data` is used and the schema will be deducted
from the passed `SerializationStrategy`.

For example, to serialize instances of the `Pizza` class:

```kotlin
@Serializable
data class Ingredient(val name: String, val sugar: Double, val fat: Double)

@Serializable
data class Pizza(val name: String, val ingredients: List<Ingredient>, val vegetarian: Boolean, val kcals: Int)

val veg = Pizza("veg", listOf(Ingredient("peppers", 0.1, 0.3), Ingredient("onion", 1.0, 0.4)), true, 265)
val hawaiian = Pizza("hawaiian", listOf(Ingredient("ham", 1.5, 5.6), Ingredient("pineapple", 5.2, 0.2)), false, 391)

val pizzaSchema = Avro.default.schema(Pizza.serializer())
val os = Avro.default.openOutputStream(Pizza.serializer()) {
 encodeFormat = AvroEncodeFormat.Binary
 schema = pizzaSchema
}.to(File("pizzas.avro"))
os.write(listOf(veg, hawaiian))
os.flush()
```

### Deserializing

We can easily deserialize a file back into data classes using instances of `AvroInputStream` which work in a similar way to the output stream version.
Given the `pizzas.avro` file we generated in the previous section on serialization, we will read the records back in as instances of the `Pizza` class. 
First create an instance of the input stream by calling `openInputStream` on an `Avro` object specifying the types we will read back.
Optionally, we can also specify a decoding format. 
Depending on the chosen format, the source and the writer schema also needs to be configured.

`AvroInputStream` has functions to get the next single value, or to return all values as an iterator.

For example, the following code:

```kotlin
@Serializable
data class Ingredient(val name: String, val sugar: Double, val fat: Double)

@Serializable
data class Pizza(val name: String, val ingredients: List<Ingredient>, val vegetarian: Boolean, val kcals: Int)

val veg = Pizza("veg", listOf(Ingredient("peppers", 0.1, 0.3), Ingredient("onion", 1.0, 0.4)), true, 265)
val hawaiian = Pizza("hawaiian", listOf(Ingredient("ham", 1.5, 5.6), Ingredient("pineapple", 5.2, 0.2)), false, 391)

val pizzaSchema = Avro.default.schema(Pizza.serializer())

val output = Avro.default.openOutputStream(Pizza.serializer()) {
 encodeFormat = AvroEncodeFormat.Binary
 schema = pizzaSchema
}.to(File("pizzas.avro"))
output.write(listOf(veg, hawaiian))
output.close()

val input = Avro.default.openInputStream(Pizza.serializer()) {
 decodeFormat = AvroDecodeFormat.Binary(/* writerSchema */ pizzaSchema, /* readerSchema */pizzaSchema)
}.from(File("pizzas.avro"))
input.iterator().forEach { println(it) }
input.close()
```

Will print out the following:

```text
Pizza(name=veg, ingredients=[Ingredient(name=peppers, sugar=0.1, fat=0.3), Ingredient(name=onion, sugar=1.0, fat=0.4)], vegetarian=true, kcals=265)
Pizza(name=hawaiian, ingredients=[Ingredient(name=ham, sugar=1.5, fat=5.6), Ingredient(name=pineapple, sugar=5.2, fat=0.2)], vegetarian=false, kcals=391)
```

### Using avro4k in your project

Gradle
```compile 'com.github.avro-kotlin.avro4k:avro4k-core:xxx'```

Maven
```xml
<dependency>
    <groupId>com.github.avro-kotlin.avro4k</groupId>
    <artifactId>avro4k-core</artifactId>
    <version>xxx</version>
</dependency>
```

Check the latest released version on Maven Central

### Known problems

Kotlin 1.7.20 up to 1.8.10 cannot properly compile @SerialInfo-Annotations on enums (see https://github.com/Kotlin/kotlinx.serialization/issues/2121). 
This is fixed with kotlin 1.8.20. So if you are planning to use any of avro4k's annotations on enum types, please make sure that you are using kotlin >= 1.8.20. 

### Contributions

Contributions to avro4k are always welcome. Good ways to contribute include:

- Raising bugs and feature requests
- Fixing bugs and enhancing the DSL
- Improving the performance of avro4k
= Adding to the documentation
