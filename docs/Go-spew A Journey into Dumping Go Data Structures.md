# Go-spew: A Journey into Dumping Go Data Structures

While it is clearly better to have access to a fancy debugger when debugging your applications, sometimes it just isn’t practical depending on the environment, remote access capabilities, and many other factors. In those cases, you just can’t beat a good old fashioned print statement to dump the state of your data structures.

Enter [go-spew](https://github.com/davecgh/go-spew), a freely available deep pretty printer package for Golang data structures to aid in debugging.

If you simply want to use the package, the documentation provided via the link above provides everything you need, but if you’re interested in seeing how the package came to life and some of the issues associated with providing a deep pretty printer, read on.

# Dumping structures in C versus Go

As anyone who has worked with Go is likely already aware, using the standard Go fmt library to print a data structure with the %v format specifier is incredibly useful. This is even more apparent if you’re accustomed to languages like C where dumping your data structures generally requires multiple printf statements for each field along with knowing a variety of format specifiers and dealing with myriad portability issues. For example, consider the following comparison of C code to portably dump a simplified version of an actual C struct used in [Cyphertite](https://www.cyphertite.com) and its equivalent Go code. As you can see, it’s hard to deny the superiority of Go when it comes to printing your data.

C:
```c
struct ctfile_header {
    int       cmh_beacon;   /* magic marker */
    int64_t   cmh_nr_shas;  /* total shas */
    uint32_t  cmh_mode;     /* file mode */
    u_char    cmh_type;     /* file type */
    char     *cmh_filename; /* original filename */
    /* Fields redacted for brevity */
};

/* Sample C dump code assuming pctfh is a pointer to existing struct */
printf("{beacon:%d", pctfh->cmh_beacon);
printf(" numshas:%"PRId64, pctfh->cmh_nr_shas);
printf(" mode:%"PRIu32, pctfh->cmh_mode);
printf(" type:%u", pctfh->cmh_type);
printf(" filename:%s}\n", pctfh->cmh_filename);

/* Sample Output */
{beacon:1297035375 numshas:82 mode:33060 type:4 filename:test}
```

Go:
```go
type ctfileHeader struct {
    beacon   int32  // magic marker
    numShas  int64  // total shas
    mode     uint32 // file mode
    typ      uint8  // file type
    filename string // original filename
}

// Sample Go dump code assuming pctfh is a pointer to existing struct
fmt.Printf("%+v\n", pctfh)

// Sample Output
&{beacon:1297035375 numShas:82 mode:33060 typ:4 filename:test}
```

# Fmt Package Limitations

As nice as the standard fmt package is, it does have some limitations when it comes to handling more complex data structures, particularly when it comes to pointers. Let me be clear here that I believe the choices made in fmt with regards to these limitations were the right decision for general purpose output in the standard libraries. Since fmt is the main method of outputting information in Go, it needs to be reasonably efficient. There are several issues that crop up when you start to dig into deep printing data structures, some of which are outlined in the following sections. Properly handling these issues adds overhead which is fine when you’re debugging, but you do not want it in the general case.

That said, the following are some limitations of the fmt package which led to the birth of [go-spew](https://github.com/davecgh/go-spew) to address them:

- Pointers are not dereferenced and followed
- Custom Stringer/error interfaces are not invoked on unexported fields
- For proper type safety, custom types which only implement the Stringer/error interfaces via a pointer receiver are not invoked on non-pointer variables

# Pointer dereferencing

To show a real world example of where dereferencing pointers and displaying them is useful, let’s build on the ctfileHeader structure shown near the beginning of the article with a highly simplified version of another actual struct used in Cyphertite C code.

```c
// Trimmed down to only the relevant pieces and a dummy field
// to illustrate the points.
type ctfileParseState struct {
    header *ctfileHeader
    bar    string
}

// Mock structs to display.
ctfh := ctfileHeader{1297035375, 82, 33060, 4, "test"}
c := ctfileParseState{&ctfh, "bar"}
```

The ordinary fmt output is not very useful due to the pointer, as can be seen in this next snippet. Compare it with the go-spew output.

```go
fmt.Printf("%+v\n", c)

// fmt.Printf Output:
{header:0xf84003f480 bar:bar}


spew.Printf("%+v\n", c)

// spew.Printf Output:
{header:<*>(0xf84003f480){beacon:1297035375 numShas:82 mode:33060 typ:4 filename:test} bar:bar}
```

Since data structures can get quite complex, go-spew also provides the spew.Dump method to provide customizable indentation and newlines:

```go
spew.Dump(c)

// spew.Dump Output:
(main.ctfileParseState) {
 header: (*main.ctfileHeader)(0xf84003f480)({
  beacon: (int32) 1297035375,
  numShas: (int64) 82,
  mode: (uint32) 33060,
  typ: (uint8) 4,
  filename: (string) "test"
 }),
 bar: (string) "bar"
}
```

# Custom Stringer/error Interfaces

```go
type flag uint16

const (
    // Several flags redacted for brevity
    flEncrypted       flag = 1 << 5
    flCompressionLzo  flag = 1 << 12
    flCompressionLzw  flag = 2 << 12
    flCompressionLzma flag = 3 << 12
    flCompressionMask flag = 0xf000
)

// Map of flags back to their constant names for pretty printing.
var flagStrings = map[flag]string{
    flEncrypted:       "flEncrypted",
    flCompressionLzo:  "flCompressionLzo",
    flCompressionLzw:  "flCompressionLzw",
    flCompressionLzma: "flCompressionLzma",
}

// String returns the flag in human-readable form.
func (fl flag) String() string {
    // No flags are set.
    if fl == 0 {
        return "0x0"
    }

    // Add individual bit flags.
    s := ""
    for flag, name := range flagStrings {
        if (fl&flag == flag) && (flag&flCompressionMask == 0) {
            s += name + "|"
            fl -= flag
        }
    }

    // Add compression type based on compression mask.
    compressionType := fl & flCompressionMask
    if name := flagStrings[compressionType]; name != "" {
        s += name
        fl -= compressionType
    }

    s = strings.TrimRight(s, "|")
    if fl != 0 {
        s += "|0x" + strconv.FormatUint(uint64(fl), 16)
    }
    s = strings.TrimLeft(s, "|")
    return s
}
```

The issue with the standard fmt package is that it can’t invoke the Stringer interface on unexported fields. The following code illustrates this issue and compares it with the go-spew output:

```go
type Foo struct {
    f  flag
    pf *flag
}
f := flEncrypted | flCompressionLzma
foo := Foo{flEncrypted | flCompressionLzo, &f}

fmt.Printf("%v\n", foo)

// fmt.Printf Output:
{4128 0xf840041020}

spew.Printf("%v\n", foo)

// spew.Printf Output:
{flEncrypted|flCompressionLzo <*>flEncrypted|flCompressionLzma}
```

Being able to see that the flEncrypted and flCompressionLzo flags are set at a glance sure beats the heck out of “4128” or, in the case of the pointer, a pointer address.

# Circular Data Structures

As can be seen above, it is undoubtedly useful to automatically follow pointers and print data they point to. However, this quickly raises questions about circular data structures. The following example code illustrates a trivial example of a Circular data structure. While this one is easy to spot, in practice, circular data structures are usually a lot more complex and happen through transitive dependencies. Go-spew detects the condition and handles it properly.

```go
type Circular struct {
        a    int
        next *Circular
}
c := &Circular{1, nil}
c.next = c

fmt.Printf("%+v\n", c)

// fmt.Printf Output:
&{a:1 next:0xf84002d200}

// spew.Printf Output:
<*>(0xf84002d200){a:1 next:<*>(0xf84002d200)<shown>}
```

# Depth Limitations

Another common question regarding deep printing data structures is how to handle deeply nested data. While there are more drastic methods such as panics, go-spew handles this situation by making the depth limit configurable rather than enforcing an arbitrary limit by default. Recall from the previous section that circular data structures are already detected and handled nicely, so infinitely unbounded depth is not an issue.

The following code assumes the same data structures introduced in the previous sections, however it limits the maximum depth to 1.

```go
ctfh := ctfileHeader{1297035375, 82, 33060, 4, "test"}
c := ctfileParseState{&ctfh, "bar"}

spew.Config.MaxDepth = 1

spew.Printf("%+v\n", c)

// spew.Printf Output:
{header:<*>(0xf84003f450){<max>} bar:bar}

spew.Dump(c)
// spew.Dump Output:
(main.ctfileParseState) {
 header: (*main.ctfileHeader)(0xf84003f450)({
  <max depth reached>
 }),
 bar: (string) "bar"
}
```

# Conclusion

Phew! This article got a little longer than I planned, but hopefully it demonstrates the usefulness and some of the challenges associated with deep pretty printing Go data structures.

If you think you would find this capability as useful as I do, check it out at github at https://github.com/davecgh/go-spew.