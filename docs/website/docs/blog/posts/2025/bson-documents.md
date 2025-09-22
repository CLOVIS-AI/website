---
date: 
  created: 2025-06-23
slug: bson-documents
tags:
  - MongoDB
---

# MongoDB documents are strange

MongoDB stores all its information in its own binary format called BSON. Lately, I've been writing my own implementation of BSON, and so I've been studying its specification. Here are a few things I learned about BSON documents.

<!-- more -->

## BSON primer

[MongoDB](http://mongodb.com/) is a document-oriented database: instead of considering *tables* with *rows* of data that all have the same structure, MongoDB considers *collections* that contain *documents* which are all independent. Multiple documents in a single collection may have a completely different shape from each other.

Of course, this flexibility has major implications on the maintenance of software, but this is a topic for another time.

What's important for today is that each document must contain enough information to understand its structure. Additionally, to be able to perform complex queries efficiently, the database must be able to find the information it wants from a document *fast*.

To achieve these goals, BSON is inspired by JSON but makes quite a few changes:

- First, BSON is a binary format instead of a textual format. This doesn't necessarily make BSON more compact than JSON (in fact, it's not rare that a BSON payload is larger than its JSON equivalent), but it does make it more previsible.
- BSON is optimized for finding specific fields easily. To do so, the format ensures that it is always trivial to know the size of a field, so we can directly skip to the next one if we're not interested, without having to parse its children.

If you want to learn more about the differences between JSON and BSON, see [the official comparison article](https://www.mongodb.com/resources/basics/json-and-bson).

Let's see how a simple JSON document can be encoded to BSON: ```#!json {"hello": "world"}```. Because raw binary is quite unreadable for humans, let's look at the hexadecimal representation of the equivalent BSON:

```text
16 00 00 00 02 68 65 6c 6c 6f 00 06 00 00 00 77 6f 72 6c 64 00 00
|---------- |  |---------------- |---------- |---------------- |
|           |  |                 |           |                 Document end
|           |  |                 |           w  o  r  l  d
|           |  |                 String size: 6 bytes
|           |  h  e  l  l  o
|           The next field is a String
Document start: 16 bytes
```

Let's compare that to the JSON example encoded in UTF-8:
```text
7b 22 68 65 6c 6c 6f 22 3a 22 77 6f 72 6c 64 22 7d
{  "  h  e  l  l  o  "  :  "  w  o  r  l  d  "  }
```

As you can see, this very simple example is more compact in JSON than BSON. Let's point out a few differences:

**In BSON, strings are null-terminated.**

:   Compare the "hello" in BSON (`68 65 6c 6c 6f 00`) versus JSON (`68 65 6c 6c 6f`).

**In BSON, string values (but not field names) are prefixed by their size in bytes.**

:   In BSON, we know that the string "world" will be 6 bytes even before reading it. If we're searching for another field, we can very efficiently skip 6 bytes to quickly find the next field.

:   Field names don't need a size because we're always forced to read them if we're searching a given field.

**Fields have a type.**

:   BSON has strings, 32-bit signed and unsigned integers, double-precision floating-point numbers, quadruple-precision floating-point numbers, strings, UTC timestamps, null, UUIDs, etc.

:   For each type, we either deterministically know its size (e.g. `int32` is always four bytes) or it is encoded directly after the field name (e.g. strings as already mentioned above). We can therefore always know, with minimal effort, how many bytes to skip to get to the next field.

:   Because documents keep their hierarchical structure, we can skip over entire sub-document hierarchies to find the field we're interested in, without parsing any of them.

**Documents are null-terminated and prefixed by their size.**

:   Just like strings, the entire document is null-terminated and starts with its size in bytes. In addition to making skipping possible, this also allows us to immediately allocate a byte array large enough to encode the entire document.

Now that we understand the basics of how BSON is encoded (go [here](https://bsonspec.org/spec.html) if you want more information), here are a few things I didn't expect.

## BSON is just documents

Any JSON value can be used at the top-level of a JSON file. Here are a few valid examples:
```json
{
	"hello": "world"
}
```
```json
[
	1,
	2,
	3
]
```
```json
"Hi!"
```
As you can see, a JSON file can contain a document, an array, or even just a string.

But that's… not the case in BSON. In BSON, the top-level is **always** a document. This clashes with the mental model of many programming language libraries, which are inspired by JSON. For example, you would expect to be able to create a BSON array and print it to test its contents—but it's not possible to print an array by itself. Similarly, it's not possible to have a variable that contains a BSON array, because a BSON entity must always be a document.

In [KtMongo](https://opensavvy.gitlab.io/ktmongo/docs), one of my goals is to provide simple debugging: all driver elements have a textual representation in JSON, derived from their BSON representation. When viewing anything in the debugger, or when logging anything, they appear as they would if the developer was making a request manually in their IDE. This allows copying the output of complex requests to edit them in an IDE to understand what each part of it does, without having to rewrite it.

But of course, many Kotlin objects are not documents. In particular, it's quite common to use BSON arrays. But the Java driver cannot represent JSON arrays! So, what to do?

??? danger "Spoiler"
    You can find my solution [here](https://gitlab.com/opensavvy/ktmongo/-/blob/82c23adf481fefc3ee4ec6aaebb5e8b0c74f3472/bson-official/src/jvmMain/kotlin/BsonReader.jvm.kt#L70-L80). Basically, it creates a temporary BSON document, adds the array to it, serializes that, then uses simple string manipulation to remove the surrounding document.

    Is this great code? No! But the official drivers don't support simpler solutions, because the spec doesn't allow arrays as valid top-level values.

Interestingly, there are multiple different places in the MongoDB protocol where multiple similar top-level documents are required. For example, we can send multiple commands to the database in a single message: [each of them is encoded as a BSON document](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/#std-label-wire-msg-sections) that just follow each other, which is not how BSON arrays are formatted—though that's a story for another time. 

## BSON documents can contain duplicate fields

Well, yes and no.

The BSON specification doesn't mention what happens if a BSON document has multiple fields with the same name. Which means it doesn't forbid them. The specification does explicitly forbid arrays that contain multiple elements at the same index, which makes it even stranger that it doesn't forbid the same of documents.

This doesn't mean duplicate fields are supported, however. In the presence of duplicate fields, tooling behaves in different ways. The following table is a summary of my experimental and very unscientific tests—please report errors if you find any!

- **MongoDB** operators (`$eq`, `$set`…) only apply to the first field for a given name.
- **MongoDB Compass** only takes into account the first occurrence. 
- **Studio3T** only takes into account the last occurrence.
- The **official Java driver** only takes into account the last occurrence.
- The **KtMongo Multiplatform** driver only writes all duplicates, but only reads the last one (though this behavior will most likely change).

Because tools hide duplicates, it is quite hard to know about them.

If you can test other tools, please send me the results and I'll add them here!
You can use the payload `1B0000000378001300000010610001000000106100020000000000`: `#!json {"x": {"a": 1, "a": 2}}`.

If you think one behavior is better than another, please [chime in here](https://gitlab.com/opensavvy/ktmongo/-/issues/66) as we're considering what the KtMongo behavior should be.

## Conclusion

BSON is an interesting format with specific tradeoffs. Specifically, BSON uses a bit more space to make scanning faster.

MongoDB users usually think in terms of JSON. BSON is quite similar, but also has a few differences. In this article, we saw that BSON has proper data types, that a BSON value can only be a document, and that documents can contain duplicate fields (though they really shouldn't).

Next time, I'll discuss BSON arrays specifically. After all, they don't exist.
