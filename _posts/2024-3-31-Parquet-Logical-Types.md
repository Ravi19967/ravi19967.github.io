---
title: 'Parquet Metadata: Deep Dive into Types'
date: 2024-03-31
permalink: /posts/2024/3/parquet-logical-types/
tags:
  - types
  - encoding
  - parquet
---
## Introduction
Expanding upon the groundwork laid in the previous [blog post]((https://rr43.net/posts/2024/2/parquet-metadata-dataserde/)) which introduced the core concepts of Parquet metadata, this subsequent entry delves even deeper into the intricate realm of metadata information and exploring the nuances of logical types. Join me on this illuminating journey as we unravel the inner workings of Parquet metadata, uncovering its profound impact on data management and analytics.

## Parquet Metadata
All parquet metadata is seralised using TCompactProtocol. There are 3 types of metadata in parquet:
- File metadata
- Column metadata
- Page header metadata

Here is a logical flow of relationship between various concepts inside parquet file:
![image](../images/Parquet-metadata-diagram.jpg)

### File Metadata
[File Metadata](https://github.com/apache/parquet-format/blob/e517ac4dbe08d518eb5c2e58576d4c711973db94/src/main/thrift/parquet.thrift#L1109-L1166) in parquet refers to the definiton of the following attributes:
- Version: This **required** field defines the version of parquet format that is being used by the tool that is writing parquet file. This is important to understand compatibility of files within in a group of files
- Schema: This **required** field defines the schema that contains the metadata for all columns. Since parquet supports nested structures. It follows a depth first search approach and converts all paths to a list of [schema element type](https://github.com/apache/parquet-format/blob/e517ac4dbe08d518eb5c2e58576d4c711973db94/src/main/thrift/parquet.thrift#L408-L468). The column metadata contains the path in the schema for that column which can be used to map columns to nodes in the schema. The first element is the root node that doesn't correspond to any data.
    ```
    ## For example if we have a data structure of the following nature
    root:
        field_1:
            field_2: 'en-us'
        field_3:
            field_4: 'en'

    ## The schema tree would look like the following:
        root
        /  \
    field_1 field_3
        /      \
    field_3   field_4
    ```
- num_rows: This **required** field defines the number of rows of data present in the file. This information can be used to return count over files in spark but depends on the processing engine.
- row_groups: This **required** field defines the number of horizontal partitioning data into rows. There is no physical structure that is guaranteed for a row group. Map Reduce operations are based on the grain of row groups.
- key_value_metadata: Key value metadata is an **optional** field that can be used to add custom information. This can be used to document your dataset
- created_by: This **optional** field provides information about the application that created the parquet file
- column_orders: This **optional** field is a list of ColumnOrder. The ColumnOrder is the basis that is used to define max & min statistics for all columns. This is important functionality as string type columns are UTF-8 encoded that don't have negative integer mappings so it makes sense to support unsigned comparision for UTF-8 based lexicographic comparison. [More Info](https://issues.apache.org/jira/browse/PARQUET-686)

### Logical Types
The types supported by the parquet file format are intended to be as minimal as possible, with a focus on how the types effect on disk storage. Logical types are used to extend these types that parquet can be used to store, by specifying how the primitive types should be interpreted. This keeps the set of primitive types to a minimum and reuses parquet's efficient encodings. For example, strings are stored as byte arrays (binary) with a UTF8 annotation. Before we jump into logical types, let's look at the primitive types we have in parquet:
- BOOLEAN
- INT32
- INT64
- INT96
- FLOAT
- DOUBLE
- BYTE_ARRAY
- FIXED_LEN_BYTE_ARRAY

There is an older representation of the logical type annotations called ConvertedType. To support backward compatibility with old files, readers should interpret LogicalTypes in the same way as ConvertedType, and writers should populate ConvertedType in the metadata according to well defined conversion rules. The reason why ConvertedType can't be extended to support LogicalType is because thrift enums don't support additional type parameters like decimal scale and precision. The new LogicalType representation is a union of structs of logical types, this way allowing more flexible API, logical types can have type parameters. The logical types can be broadly put into the following categories:
- String types
- Numeric types
- Temporal types
- Embedded Types
- Nested Types
- UNKNOWN

#### String types
- STRING: STRING may only be used to annotate the binary primitive type and indicates that the byte array (BYTE_ARRAY) should be interpreted as a UTF-8 encoded character string.
- ENUM: ENUM annotates the binary primitive type (BYTE_ARRAY) and indicates that the value was converted from an enumerated type in another data model (e.g. Thrift, Avro, Protobuf).
- UUID: UUID annotates a 16-byte fixed-length binary (FIXED_LEN_BYTE_ARRAY).

#### Numeric types
- Signed Integers: The INT annotation provides a means to precisely specify the maximum number of bits used in the stored integer values. It consists of two parameters: bit width and sign. Acceptable bit width values include 8, 16, 32, and 64, while the sign parameter can be set to either true or false. When indicating signed integers, the second parameter should be set to true. For instance, a signed integer with a bit width of 8 is defined as INT(8, true). Utilizing these annotations, implementations can optimize memory usage by employing smaller in-memory representations when reading data.
- Unsigned Integers: The INT annotation serves as a versatile tool for specifying both unsigned integer types and the maximum number of bits utilized in the stored value. It encompasses two essential parameters: bit width and sign. Permissible bit width values encompass 8, 16, 32, and 64, while the sign parameter can be toggled to either true or false. Specifically for unsigned integers, the second parameter should be set to false. For instance, defining an unsigned integer with a bit width of 8 would be denoted as INT(8, false). Leveraging these annotations, implementations can optimize memory utilization by adopting more compact in-memory representations during data retrieval.
- DECIMAL: DECIMAL annotation represents arbitrary-precision signed decimal numbers of the form unscaledValue * 10^(-scale).
- FLOAT16: The FLOAT16 annotation represents half-precision floating-point numbers in the 2-byte IEEE little-endian format.

#### Temporal types
- DATE: DATE is used to for a logical date type, without a time of day. It must annotate an int32 that stores the number of days from the Unix epoch, 1 January 1970.
- TIME: TIME is used for a logical time type without a date with millisecond or microsecond precision. The type has two type parameters: UTC adjustment (true or false) and unit (MILLIS or MICROS, NANOS).
- TIMESTAMP: In data annotated with the TIMESTAMP logical type, each value is a single int64 number that can be decoded into year, month, day, hour, minute, second and subsecond fields using calculations detailed below. Please note that a value defined this way does not necessarily correspond to a single instant on the time-line and such interpretations are allowed on purpose. The TIMESTAMP type has two type parameters:
    - isAdjustedToUTC must be either true or false.
    - unit must be one of MILLIS, MICROS or NANOS. This list is subject to potential expansion in the future. Upon reading, unknown unit-s must be handled as unsupported features (rather than as errors in the data files).
    - if one wants to store 1970-01-03 00:00:00 (UTC+01:00) as a `TIMESTAMP(isAdjustedToUTC=true, unit=MILLIS)`, first the time zone offset has to be dealt with. By normalizing the timestamp to UTC, we calculate what time in UTC corresponds to the same instant: 1970-01-02 23:00:00 UTC. This is 1 day and 23 hours after the epoch, therefore it can be encoded as the number (24 + 23) * 60 * 60 * 1000 = 169200000. Please note that time zone information gets lost in this process. Upon reading a value back, we can only reconstruct the instant, but not the original representation. In practice, such timestamps are typically displayed to users in their local time zones, therefore they may be displayed differently depending on the execution environment.
    - Using a single number to represent a local timestamp is a lot less intuitive than for instants. One must use a local timestamp as the reference point and calculate the elapsed time between the actual timestamp and the reference point. The problem is that the result may depend on the local time zone, for example because there may have been a daylight saving time change between the two timestamps. The solution to this problem is to use a simplification that makes the result easy to calculate and independent of the timezone. Treating every day as consisting of exactly 86400 seconds and ignoring DST changes altogether allows us to unambiguously represent a local timestamp as a difference from a reference local timestamp. We define the reference local timestamp to be 1970-01-01 00:00:00 (note the lack of UTC at the end, as this is not an instant). This way the encoding of local timestamp values becomes very similar to the encoding of instant values. For example, in a TIMESTAMP(isAdjustedToUTC=false, unit=MILLIS), the number 172800000 corresponds to 1970-01-03 00:00:00 (note the lack of UTC at the end), because it is exactly two days from the reference point (172800000 = 2 * 24 * 60 * 60 * 1000).
- INTERVAL: INTERVAL is used for an interval of time. It must annotate a fixed_len_byte_array of length 12. This array stores three little-endian unsigned integers that represent durations at different granularities of time. The first stores a number in months, the second stores a number in days, and the third stores a number in milliseconds. This representation is independent of any particular timezone or date. The sort order used for INTERVAL is undefined. When writing data, no min/max statistics should be saved for this type.

#### Embedded Types
Embedded types do not have type-specific orderings and hence are rarely uses. Usually you would define a nested schema if you have complex data structure. If you use these mentioned types, they would store the result in binary string.
- JSON: JSON is used for an embedded JSON document. It must annotate a binary primitive type. [JSON specification](http://json.org/)
- BSON: BSON is used for an embedded BSON document. It must annotate a binary primitive type. [BSON specification](http://bsonspec.org/spec.html)

#### Nested Types
Here we discuss how LIST and MAP are used to encode nested types by adding group levels around repeated fields that are not present in the data.
- LIST: LIST always has a 3-level structure
    - Generic structure:
        ```
        <list-repetition> group <name> (LIST) {
        repeated group list {
            <element-repetition> <element-type> element;
            }
        }
        ```
    - The outer-most level is a group annotated with LIST that contains a single field named list. The repetition of this level is either optional or required and determines whether the list is nullable.
    - The middle level, named list, is a repeated group with a single field named element.
    - The element field encodes the list's element type and repetition. Element repetition is either required or optional.
    - Example list:
        ```
        nullable list of List<List<Integer>> with non-nullable values
        optional group array_of_arrays (LIST) {
        repeated group list {
            required group element (LIST) {
            repeated group list {
                required int32 element;
                    }
                }
            }
        }
        ```
- MAPS: MAPS should be interpreted as a map from keys to values. It has a 3-level structure:
    - Generic structure:
        ```
        <map-repetition> group <name> (MAP) {
            repeated group key_value {
                required <key-type> key;
                <value-repetition> <value-type> value;
            }
        }
        ```
    - The outer-most level must be a group annotated with MAP that contains a single field named key_value. The repetition of this level must be either optional or required and determines whether the list is nullable.
    - The middle level, named key_value, must be a repeated group with a key field for map keys and, optionally, a value field for map values.
    - The key field encodes the map's key type. This field must have repetition required and must always be present.
    - The value field encodes the map's value type and repetition. This field can be required, optional, or omitted.
    - Example non-null map from strings to nullable integers:
        ```
        // Map<String, Integer>
        required group my_map (MAP) {
            repeated group key_value {
                required binary key (UTF8);
                optional int32 value;
            }
        }
        ```
    - If there are multiple key-value pairs for the same key, then the final value for that key must be the last value. This is an important point, that means in parquet we read all keys before we decide on the value to infer, which can have a bit of cost when reading hundereds of keys.

#### UNKNOWN (always null)
In scenarios where the schema of existing data lacks type information and all values within a column are consistently null, the UNKNOWN type comes into play. Similar to the Null type found in Avro and Arrow, UNKNOWN serves as an annotation for columns where all values are null. This annotation helps maintain schema consistency and provides a clear indication of the absence of meaningful data in that column.


## Takeaway
In this blogpost we learnt, Parquet metadata plays a pivotal role in organizing and interpreting structured data efficiently.

### Parquet Metadata
- File Metadata: Crucial information defining the structure and attributes of Parquet files, including version, schema, number of rows, and row groups.
- Logical Flow: Visualization illustrating the relationship between different metadata concepts within Parquet files.
- Column Orders: Optional field aiding in defining max & min statistics for columns, facilitating optimized data processing.
- Key-Value Metadata: Optional custom metadata for additional documentation and contextual information.
- Created By: Optional field providing insights into the application responsible for creating the Parquet file.

### Logical Types
Logical types extend the capabilities of Parquet's primitive types, enabling more nuanced interpretations of data. Here's a breakdown of logical type categories and their significance:

- String Types: Annotations like STRING, ENUM, and UUID enrich byte array interpretations, enhancing string data handling.
- Numeric Types: Precisely specify integer and floating-point representations, including support for decimals and half-precision floats.
- Temporal Types: Facilitate accurate storage and interpretation of dates, times, timestamps, and intervals, catering to diverse temporal data requirements.
- Embedded Types: Allow embedding JSON and BSON documents within Parquet files, expanding data flexibility.
- Nested Types: Utilize LIST and MAP structures to encode nested data efficiently, enabling complex data representations.
- UNKNOWN Type: Indicates columns where all values are consistently null, maintaining schema consistency and clarity.

In the next blogpost we will talk about encoding this information and encryption of data. Stay tuned for more.