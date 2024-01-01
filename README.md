
# DropXML

A simple, header-only C++ XML parser.

## How to use

Simply *drop* the header file ([dropXML.hpp](dropXML.hpp)) into your project and include it.
To parse an XML file, you have to load it into a string of consecutive `char`s and implement a class that consumes the XML data. It has the following API:

```c++
/// Receives data from XML parser. All functions should return true to continue parsing, false to abort.
class MyXmlConsumer {

public:
    /// Called when a processing instruction like <?xml is encountered.
    bool processingInstruction(const char *nameStart, const char *nameEnd);
    /// Called when a <!DOCTYPE definition is encountered. The entire string between <!DOCTYPE and > is passed, without parsing its content.
    bool doctype(const char *doctypeStart, const char *doctypeEnd);
    /// Called when entering an <element> with the specified name.
    bool enterElement(const char *nameStart, const char *nameEnd);
    /// Called when leaving an </element> with the specified name.
    bool leaveElement(const char *nameStart, const char *nameEnd);
    /// Called when encountering an attribute name="value" of the last-entered element or processing instruction. Call decode on value.
    bool elementAttribute(const char *nameStart, const char *nameEnd, const char *valueStart, const char *valueEnd);
    /// Called when all attributes have been submitted and before the element's content.
    bool finishAttributes();
    /// Called when a text string is encountered. Leading and trailing whitespace is removed. Call decode on the string.
    bool text(const char *textStart, const char *textEnd);
    /// Called when a CDATA (character data) block is encountered. Do not call decode on the string.
    bool cdata(const char *cdataStart, const char *cdataEnd);
    /// Called at the end of the XML file.
    bool finish();

};
```

If you are not interested in some specific XML features like a `<!DOCTYPE>` specification, simply ignore the passed values and return true.

With that, just call `dropXML::parse<MyXmlConsumer>` with the pointer to the XML string's beginning and end.
For text and attribute values, you should call `dropXML::decode` to resolve any XML entities like `&amp;`.
Here is an example of how to use it to make a `std::string`:

```c++
std::string xmlDecode(const char *start, const char *end) {
    // Check if a buffer is needed
    if (!dropXML::decode(start, end, nullptr, nullptr)) {
        // Allocate buffer string at least as long as the input sequence
        std::string buffer(end-start+1, '\0');
        // Decode input into buffer string
        if (!dropXML::decode(start, end, &buffer[0], &buffer[buffer.size()-1]))
            return std::string();
        if (start == &buffer[0]) {
            // Shorten buffer string to actual size
            buffer.resize(end-start, '\0');
            return buffer;
        }
    }
    return std::string(start, end);
}
```

## Is this the right XML parser for me?

*DropXML* is very straight-forward to integrate and use, but this comes with both pros & cons:

- **Compatibility** - the parser implementation has *no dependencies*, not even standard library includes. It does not rely on anything platform-specific, performs *no allocation*, and does not use any modern C++ features. It currently cannot be used in plain C because of its use of templates and classes but it could be easily modified to remove those.
- **Performance** - the parser does not perform any computationally intensive tasks so it should be fairly performant, but it largely depends on your consumer class. By itself, it does not build a DOM or any other internal structures. A template class is used instead of function pointers or virtual calls to minimize overhead.
- **Input format** - the entirety of the XML data must be provided as a consecutive array of 8-bit characters (`char`s). It does not have to be null-terminated. No other format, like UTF-16 or passing the data in chunks, is supported.
- **All-at-once** - the only way to parse an XML file with this library is to do it in one go. You can interrupt it in the middle and use the data up to that point, but you cannot resume it.
- **No DOM, no lookup, no advanced logic** - it only takes care of the XML syntax, the rest is up to you. If you need to know the value of a specific attribute before processing an element, you need to store it yourself.
- **No validation** - the purpose of this library is to parse files that are assumed to be valid, not to validate them or perform error correction. If invalid XML is supplied, the parser might ignore small inaccuracies, but generally it will report failure with no additional details.
- **No serialization** - the library does not provide any functionality to emit XML, it is strictly a parser.
- **Advanced XML features** - CDATA blocks are supported. `<!DOCTYPE>` declaration is recognized and reported to the consumer class but its content is not taken into consideration by the parser. Same with `<?xml` `encoding` and other attributes.
- **Output encoding** - the provided `decode` function only converts XML entities like `&#x1f408;` into UTF-8 sequences, but you can use your own version instead. The rest of the parser only recognizes basic ASCII characters with special meaning in XML and is otherwise encoding-agnostic.
