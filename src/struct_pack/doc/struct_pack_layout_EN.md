# memory layout of struct_pack

**By Default**, the memory layout of struct_pack is shown in the figure bellow:

![](./images/layout/memory_layout1.svg)

It contains 4 bytes of `type hash`, followed by the data payload of variable length depends on the object provided by user.

The default memory layout applies when **both** of the following conditions are met:
1. struct_pack does not contain the compatible field
2. struct_pack does not store full type information
3. the maximum length of struct_pack's container is less than 256

In the **default case**, struct_pack could omit meta information for a more compact memory layout.

However, in the **Non-default case**, struct_pack must contain the **meta information** field, at which point the complete memory layout is shown as below:

![](./images/layout/memory_layout2.svg)

The layout of each field will be described in the following sections.

# 1. type hash(Mandatory)

The type hash is located in the header of struct_pack and is 4 bytes in total. It is used for type-checking of struct_pack.

Higher 31 bits: type hash data. It is the higher 31 bit of the MD5 hash result of the struct_pack type information.
    
Lower 1 bit： meta information flag

![](./images/layout/hash_info.svg)

# 2. meta information(Optional)
When meta information flag in type hash is 1,  struct_pack will contain the meta-information field. The meta-information consists of a one-byte meta-information header and a number of bytes of meta-information.


## 2.1 header

The meta information header is one byte in total, which describes the configuration of one struct_pack object.

The detailed layout is shown figure in bellow:

![](./images/layout/meta_info.svg)


Current configurations are:

| Configuration | Contents                                                                                       | Bit location | Dependency                                                 | Configurable |
| ------------- | ---------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------- | ------------ |
| DATA_SIZE     | DATA_SIZE represents the total bytes of the serialization result, when compatible fields exist | 0b000000**   | depends on compatible field                                | N            |
| ADD_TYPE_INFO | A flag indicates if full type information exists                                               | 0b00000*00   | Release(0) Debug(1); Could be set by user manually         | Y            |
| LEN_SIZE      | Number of bytes occupied by the containers                                                     | 0b000**000   | Depends on the latest data container in the encoded result | N            |

1. `DATA_SIZE`(2 bits). When `compatible<T>` fields are present, we need to record the total data length. so
- 00 : total data length field not present(so that no compatible fields)
- 01 : total data length is recorded in a data field of 2 bytes
- 10 : total data length is recorded in a data field of 4 bytes
- 11 : total data length is recorded in a data field of 9 bytes

2. `ADD_TYPE_INFO`(1 bit). It is a flag indicating if type information is present
- 0 : full type information not present (0 in release mode)
- 1 : full type information present

3. `LEN_SIZE` (2 bits). It records the maximum number of bytes of the containers.
- 00 : 1 byte  (in range of uint8_t, 0~256)
- 01 : 2 bytes (in range of uint16_t)
- 10 : 4 bytes (in range of uint32_t)
- 11 : 8 bytes (in range of uint64_t)

4. `RESERVED`(3 bits).

## 2.2 total data length(Optional)

The length of this field depends on `DATA_SIZE`(2/4/8bytes). It is a unsigned integer represents the total length of the struct_pack data.

## 2.3 full type information(Optional)

This field is a '\0' tailed string, which represents one legal struct_pack type. The full type information is recorded by this string and the MD5 hash value 
of this field is used to get the `type hash` higher 31 bits.

The length of this field could be determined at compile time if it passed validation. However, it's length could only be determined by the position of char `'\0'` when validation failed.

The encoding rules for this field and its mapping to the struct_pack type tree can be found in[struct_pack的类型系统]。


Take the following type as an example:
```cpp
struct person {
  int age;
  std::string name;
};
```
Its type info is as follows:

![类型信息](./images/layout/person_type_info.svg)

We can see that type info ends with `\0`.


# 3. known data payload(Mandatory)

Serialization results of all known data fields could be found in this payload.

## 3.1 Structural data

Structural data types(such as struct, class, pair, tuple and so on) is a data container of defined data field. In struct_pack, all data fields in one struct are Sequential placed in the memory in the order of their definition in struct, and no more additional information required in this kind of memory layout. 

For example, if we define a struct person as bellow:
```cpp
struct person {
  int age;
  std::string name;
};
```
In release mode, the serialization of one `person{.age=24,.name="Betty"}` object could be:


![person in release mode](./images/layout/release_person.svg)

No meta information in the figure above because it's zeros and could be omitted by default.

However, in debug mode, additional meta information will be inserted into the serialization result to avoid hash conflicts, as shown bellow:

![person in debug mode](./images/layout/debug_person.svg)

As demonstrated, the meta header is `0x04`, which indicates type information is available.

And serialization of one `person{.age=24,.name=std::string{256,'A'}}` object in release mode is:

![](./images/layout/release_person_long_name.svg)
where the meta information header is `0x08`, which indicates a `size_type` length of `2` bytes.



## 3.2 fundamental types

All fundamental types(such as signed and unsigned integers, floating-point types, character types, enum and boolean) have definitive length.

### fix-sized unsigned integers


| type     | length(bytes) | format        |
| -------- | ------------- | ------------- |
| uint8_t  | 1             | original code |
| uint16_t | 2             | original code |
| uint32_t | 4             | original code |
| uint64_t | 8             | original code |


### fix-sized unsigned integers

| type    | length(bytes) | format           |
| ------- | ------------- | ---------------- |
| int8_t  | 1             | two's Complement |
| int16_t | 2             | two's Complement |
| int32_t | 4             | two's Complement |
| int64_t | 8             | two's Complement |

### fix-sized unsigned integers


| type                      | length(bytes)        | format |
| ------------------------- | -------------------- | ------ |
| struct_pack::var_uint32_t | 1-5 (variant length) | varint |
| struct_pack::var_uint64_t | 1-10(variant length) | varint |

### variant length signed integers


| type                     | length(bytes)        | format          |
| ------------------------ | -------------------- | --------------- |
| struct_pack::var_int32_t | 1-5 (variant length) | zigzag + varint |
| struct_pack::var_int64_t | 1-10(variant length) | zigzag + varint |


### fix-sized floating-point types

| type   | length(bytes)) | format                    |
| ------ | -------------- | ------------------------- |
| float  | 4              | IEEE 754 single-precision |
| double | 8              | IEEE 754 double-precision |


### fix-sized character types


| type     | length(bytes))     | format        |
| -------- | ------------------ | ------------- |
| char8_t  | 1                  | original code |
| char16_t | 2                  | original code |
| char32_t | 4                  | original code |
| wchar_t  | Platform-dependent | original code |


### enum

Enum types shares the same memory layout with the corresponding integer types.

### boolean

Boolean takes on byte. It is `false` if all bits of this byte are 0 and true otherwise.

## 3.3 data containers

Most data structures are supported in struct_pack and each of them has its dedicated memory layout.

### fixed-length arrays

For C-style array, C++ `std::array` or user-defined arrays, array length information is not stored because it is known in compile time. All elements are encoded by individually coding the elements of the array in their natural order, 0 through n -1.

For instance, serialization of `std::array<int,2>{24,42}` is encoded as the figure bellow:

![](images/layout/array_layout.svg)

### sequential containers

For Sequential containers such as `std::vector`,`std::deque`,`std::list`,`std::string` or any user-defined structures, the serialization begins with the number of elements of `size_type`, then all elements are encoded in their natural order.

For example, an `std::string{"Hello"}` could be encoded as bellow:

![](images/layout/string_layout.svg)

### set

For `std::set` or user-defined set, the memory layout begins with the elements count of `size_type`, followed by the numbers of `keys` in this set.

For example, a `std::set<int>{42,24}` could be encoded as:

![](images/layout/set_layout.svg)

### map

For `std::map<K,V>` or similar user-defined data, the memory layout begins with number of elements of type `size_type`, followed by each data field of `value_type`.

For example, a 

```cpp
std::map<int,std::string>{{42,"Hello"},{24,"Student"}}
```

is encoded as:

![](images/layout/map_layout.svg)


### `optional<T>`

For `std::optional<T>` or similar user-defined data, it begins with a bool indicates if the value is present or not. The value will be followed by this bool field if it is present.

For example, `std::optional<int>{42}` is encoded as:

![](images/layout/optional_has_value.svg)

And a `std::optional<int>{std::nullopt}` is encoded as:

![](images/layout/optional_null.svg)

### variant

For `std::variant` or similar user-defined data, it begins with one byte carrying the `variant index` information, of unsigned integer type, indicating the real type stored in it. Then it is followed by the encoded data of the real data type. 

For example, `std::variant<int,std::string,double>{3.14}` is encoded as:

![](images/layout/variant.svg)

### expected<T,E>

For `std::expected<T,E>` or similar user-defined data, it begins with a boolean type. When the header is true, the `expected` value is present so the memory layout is followed by the encoded value `T`. When the header is false, the error code is present so it is followed by the encoded error code.

For example, `std::expected<int,std::errc>{42}` is encoded as:

![](./images/layout/variant.svg)

and `std::expect<int,std::errc>{std::unexpected{std::errc::no_buffer_space}}` is encoded as bellow:

![](./images/layout/variant_with_error.svg)

### monostate

It is an empty type and encoded as zero length.

# 4. Unknown fields(Optional)

For backward/forward compatibility, new added fields are of type `struct_pack::compatible<T>`  be appended at the end of the object. If `compatible<T>` is present, the total data size **must** be stored in the meta information.

During deserialization, unknown `compatible<T>` fields are omitted and skipped. For `compatible<T>` fields not present, it is desensitized as empty value.

Each `compatible<T>` are placed in the same order as they are defined in the structure. And each `compatible<T>` field is encoded in the same way as `optional<T>` field.s

For example, 
```cpp
struct person {
  int age;
  std::string name;
  struct_pack::compatible<double> salary;
};
```
In release mode, object `person{.age=24,.name="Betty",.salary=2000.0}` is encoded as:

![](./images/layout/release_person_with_compatible.svg)

In the figure above, the meta information header is `0x01`, which means `compatible<T>` is contained and the total length is less than 65536.
