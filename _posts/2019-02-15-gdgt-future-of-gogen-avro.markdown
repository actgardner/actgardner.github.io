---
layout: post
title:  "GADGT: The futue of gogen-avro"
date:   2019-02-15 00:00:00
categories: golang avro vm gadgt go deserializer
---

In March 2016 I wrote [a blog post](http://www.agardner.me/golang/avro/code-generation/performance/benchmark/encoding/hadoop/kafka/2016/03/31/goavro-generator.html) about gogen-avro- a project to generate Golang codecs for serializing and deserializing data in the Apache Avro format. Over the past three years the project has been more successful than I could have hoped for: over a hundred stars, contributions from 17 people (amazing!) and several companies are using it in their data pipelines. As time has gone on I've tried to adapt the codebase to suit people's needs and better improve compatibility with the Avro spec. There has been one major sticking point: schema evolution.

As outlined in [Martin Kleppman's great blog post](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html), Avro is an untagged format where you need the reader and writer schemas to deserialize data. You can add, modify, remove and reorder fields between schema versions, and the rules are fairly flexible. To implement them all in generated code would produce giant balls of mud - giant, incomprehensible blobs of code that hurt performance because they would have to evaluate every possible evolution rule for every incoming record. I've taken several stabs at rewriting gogen-avro in this way over the past 3 years, but none of them were satisfactory.

At PyCon 2018 I was inspired by [a talk by Dema Abu Adas](https://2018.pycon.ca/talks/talk-A-9013/) to try a different approach. Instead of coding every possible evolution case into the generated Golang code, what if I implemented a VM that could handle the basic operations of deserializing data (reading, setting fields) and a compiler that could compare two schemas and write the deserializer in bytecode for that VM? 

After 3 months I'm happy to report that [GADGT (the Gogen Avro Deserializer Generating Toolkit)](https://github.com/actgardner/gogen-avro/tree/gadgt) is ready for beta testing! GADGT will be the basis of gogen-avro v6.0, which will launch in the first half of 2019. In addition to passing the existing gogen-avro suite of tests, GADGT adds support for schema evolution and reading Avro's Object Container Files seamlessly. This brings gogen-avro very close to meeting the entire Avro specification, and it should make life easier for all users. GADGT also maintains the performance of the generated code from earlier versions of gogen-avro.

The GADGT toolchain has several components:

- The gogen-avro tool still generates Golang structs for complex Avro types like unions, records, etc. These structs are designed to inter-operate with the GADGT instruction set to support retrieving fields, setting defaults and assigning values.

- The `compiler` package takes two Avro schemas (the reader and writer) and builds an intermediate representation (IR) of the deserializer. The IR contains the entire GADGT VM instruction set, but it also has instructions for deserializing blocks (for maps and arrays), switches (for unions) and methods to deserialize records (to handle recursive records). At compilation time we can also throw an error if two schemas are incompatible, according to the Avro spec.

- Once the IR is built and we know the schemas are compatible, we can build out final output bytecode. The GADGT VM only supports jumps to absolute offsets within the program, so the compiler has to map any flow control like case statements into a series of conditional jumps. Methods are all appended together, starting with `main`, and method calls are turned into pairs of call/return instructions with absolute offsets.

- Finally, the compiled bytecode can be evaluated by the VM. The VM has two stacks - one for method calls, and one for data. The data stack has a field for every different supported data type, and a new data stack frame is allocated every time we "enter" a new field (either by referring to a record field, appending to an array or inserting into a map).

As an example, in the complex-union test case (complex because the union is of complex types like map, array and record), we have this schema:

```
{
        "type": "record",
        "name": "ComplexUnionTestRecord",
        "fields": [
                {
                        "name": "UnionField",
                        "type": [
                                "null",
                                {"type": "array", "items": "int"},
                                {"type": "map", "values": "int"},
                                {"type": "record", "name": "NestedUnionRecord", "fields":[{"name": "IntField", "type": "int"}]}
                        ]
                }
        ]
}
```

And the corresponding bytecode (with comments):

```
0:      call(2) 			// The entrypoint, call into the record deserializer at offset 2
1:      halt(0) 			// halt(0) stops the VM without signalling an error
2:      enter(0) 			// This is the beginning of the record deserializer. Enter the first field
3:      |  read(long) 			// Since the field is a union, read the Long with the union type
4:      |  eval_equal(0) 		// Check the union type and jump to the appropriate deserializer 
5:      |  cond_jump(14)
6:      |  eval_equal(1)
7:      |  cond_jump(19)
8:      |  eval_equal(2)
9:      |  cond_jump(39)
10:     |  eval_equal(3)
11:     |  cond_jump(60)
12:     |  halt(1)           		// If we fall through (an unexpected union type), throw an error
13:     |  jump(65) 
14:     |  add_long(0)       		// Adjust the union value - the reader and writer unions may have different values 
15:     |  set(long)			// Set the union value
16:     |  enter(0)			// Enter the first field of the union struct, which is Null
17:     |  exit()			// Exit, we don't need to do anything for Null types
18:     |  jump(65)			// Jump past the rest of the cases
19:     |  add_long(0)
20:     |  set(long)
21:     |  enter(1)			// Enter the first field of union struct, an array of ints
22:     |  |  read(long)		// Read the length of the block
23:     |  |  eval_equal(0)		// If it's zero, we're done reading the array
24:     |  |  cond_jump(37)
25:     |  |  eval_greater(0)		// If it's negative, we need to read the byte length of the block
26:     |  |  cond_jump(29)
27:     |  |  read(UnusedLong)
28:     |  |  mult_long(-1)
29:     |  |  append_array(0)		// Append a new element to the array and read the next int into it
30:     |  |  |  read(int)
31:     |  |  |  set(int)
32:     |  |  exit(0)
33:     |  |  add_long(-1)		// Decrement the count of how many elements are in the block
34:     |  |  eval_equal(0)		// If the block is over, read the next block header
35:     |  |  cond_jump(22)
36:     |  |  jump(29)			// Otherwise, read the next element
37:     |  exit()
38:     |  jump(65)
39:     |  add_long(0)
40:     |  set(long)
41:     |  enter(2)
42:     |  |  read(long)		// Maps use the same block encoding as arrays
43:     |  |  eval_equal(0)
44:     |  |  cond_jump(58)
45:     |  |  eval_greater(0)
46:     |  |  cond_jump(49)
47:     |  |  read(UnusedLong)
48:     |  |  mult_long(-1)
49:     |  |  read(string)		// Every map value is a string, read that first
50:     |  |  append_map(0)
51:     |  |  |  read(int)
52:     |  |  |  set(int)
53:     |  |  exit(0)
54:     |  |  add_long(-1)
55:     |  |  eval_equal(0)
56:     |  |  cond_jump(42)
57:     |  |  jump(49)
58:     |  exit()
59:     |  jump(65)
60:     |  add_long(0)
61:     |  set(long)
62:     |  enter(3)
63:     |  |  call(67)			// The third type is a record. We generate a method for it at offset 67
64:     |  exit()
65:     exit()
66:     return()			// This is the end of the top-level record deserializer method
67:     enter(0)
68:     |  read(int)
69:     |  set(int)
70:     exit()
71:     return() 			// This is the end of the NestedUnionRecord deserializer method
```

Hopefully it's easy to see how this architecture makes schema evolution more tractable - it's easy to change the target field number, for instance, when fields are re-ordered. At the same time, we can compile the deserializer once and amortize the cost over many records.

I would ask anyone who has time and uses gogen-avro to try using the gadgt branch and [report any issues you experience](https://github.com/actgardner/gogen-avro/tree/gadgt#reporting-issues). I'm continuing to collect use cases and build more integration tests with real-world schemas, including multiple evolved versions of schemas. In the future I'll be writing a follow-up post about performance - right now the emphasis for GADGT is on correctness and spec-compliance.
