---
layout: post
title:  "Generating Go codecs for Avro schemas"
date:   2016-03-31 00:00:00
categories: golang avro code-generation performance benchmark encoding hadoop kafka
---

I normally use LinkedIn's excellent [goavro library](https://github.com/linkedin/goavro) to encode and decode Avro records in Go - for example, when encoding events to publish into Kakfa. LinkedIn's library is widely used and supports the entire Avro 1.7.7 standard, so it's a natural choice. However, the API is a little cumbersome. For example, to encode a record it's necessary to create a new `Record` struct and call `Set(fieldName string, value interface{}) error` repeatedly. This is problematic for a few reasons:

- every field being set has to be validated against the schema
- every value is cast to an `interface{}`, and then recast when it's encoded
- setting a map field in the record requires the value is of type `map[string]interface{}`, which requires re-casting every value
- arrays also need to be of the type `[]interface{}`, which required re-casting every element
- field types are only enforced at runtime

For example, given this record schema:

```
{
        "type": "record",
        "name": "PrimitiveTestRecord",
        "fields": [
                {"name": "IntField", "type": "int"},
                {"name": "LongField", "type": "long"},
                {"name": "FloatField", "type": "float"},
                {"name": "DoubleField", "type": "double"},
                {"name": "StringField", "type": "string"},
                {"name": "BoolField", "type": "boolean"},
                {"name": "BytesField", "type": "bytes"}
        ]
}
```

Encoding a single record requires the following method calls:

```
	someRecord.Set("IntField", 1)
        someRecord.Set("LongField", 2)
        someRecord.Set("FloatField", 3.4)
        someRecord.Set("DoubleField", 5.6)
        someRecord.Set("StringField", "789")
        someRecord.Set("BoolField", true)
        someRecord.Set("BytesField", []byte{1, 2, 3, 4})
```

A microbenchmark on my laptop takes about 4400ns per record to set the fields and encode it into a `bytes.Buffer`.

By way of comparison, I've been working on a project - [gogen-avro](https://github.com/alanctgardner/gogen-avro) - which generates Go structs and codecs from an Avro schema. For the above schema, it generates the following:

```
type PrimitiveTestRecord struct {
        IntField    int32
        LongField   int64
        FloatField  float32
        DoubleField float64
        StringField string
        BoolField   bool
        BytesField  []byte
}

func (r PrimitiveTestRecord) Serialize(w io.Writer) error {
        return writePrimitiveTestRecord(r, w)
}

func writePrimitiveTestRecord(r PrimitiveTestRecord, w io.Writer) error {
        var err error
        err = writeInt(r.IntField, w)
        if err != nil {
                return err
        }
        err = writeLong(r.LongField, w)
        if err != nil {
                return err
        }
        err = writeFloat(r.FloatField, w)
        if err != nil {
                return err
        }
        err = writeDouble(r.DoubleField, w)
        if err != nil {
                return err
        }
        err = writeString(r.StringField, w)
        if err != nil {
                return err
        }
        err = writeBool(r.BoolField, w)
        if err != nil {
                return err
        }
        err = writeBytes(r.BytesField, w)
        if err != nil {
                return err
        }

        return nil
}

// Some boilerplate methods to write primitive types

```

A microbenchmark of this code takes about 600ns per record to set the fields - a 7x speedup! Populating the struct and calling `r.Serialize()` is also much more readable, and benefits from compile-time checking.

Besides primitive types, gogen-avro can also handle arrays, maps, and union fields. Unions are particularly hairy because we want to avoid type casting and `interface{}`. A particularly large union results in code like this:

```
type UnionIntStringFloatDoubleLongBoolNull struct {
    // All the possible types the union could take on
    Int                int32
    String             string
    Float              float32
    Double             float64
    Long               int64
    Bool               bool
    Null               interface{}
    // Which field actually has data in it
    UnionType          UnionIntStringFloatDoubleLongBoolNullTypeEnum
}

// These names are obscenely long to guarantee uniqueness
type UnionIntStringFloatDoubleLongBoolNullTypeEnum int

const (
    UnionIntStringFloatDoubleLongBoolNullTypeEnumInt                UnionIntStringFloatDoubleLongBoolNullTypeEnum = 0
    UnionIntStringFloatDoubleLongBoolNullTypeEnumString             UnionIntStringFloatDoubleLongBoolNullTypeEnum = 1
    UnionIntStringFloatDoubleLongBoolNullTypeEnumFloat              UnionIntStringFloatDoubleLongBoolNullTypeEnum = 2
    UnionIntStringFloatDoubleLongBoolNullTypeEnumDouble             UnionIntStringFloatDoubleLongBoolNullTypeEnum = 3
    UnionIntStringFloatDoubleLongBoolNullTypeEnumLong               UnionIntStringFloatDoubleLongBoolNullTypeEnum = 4
    UnionIntStringFloatDoubleLongBoolNullTypeEnumBool               UnionIntStringFloatDoubleLongBoolNullTypeEnum = 5
    UnionIntStringFloatDoubleLongBoolNullTypeEnumNull               UnionIntStringFloatDoubleLongBoolNullTypeEnum = 6
)
```

gogen-avro has a bunch of limitations right now. It doesn't support:

- enumerations or fixed fields
- decoding records 
- setting the Go package name
- container formats

In general it also hasn't been tested very thoroughly - we only round-trip test primitives, arrays and maps right now. It's probably not ready for production use, especially if you produce submarines or nuclear reactors. But it'd be awesome to have more contributions and schemas to test with! [Contribute issues/PRs on Github](https://github.com/alanctgardner/gogen-avro).
