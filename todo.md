# TODO


API LIST
```
spew.Dump(myVar1, myVar2, ...)
spew.Fdump(someWriter, myVar1, myVar2, ...)
str := spew.Sdump(myVar1, myVar2, ...)

spew.Printf("myVar1: %v -- myVar2: %+v", myVar1, myVar2)
spew.Printf("myVar3: %#v -- myVar4: %#+v", myVar3, myVar4)
spew.Fprintf(someWriter, "myVar1: %v -- myVar2: %+v", myVar1, myVar2)
spew.Fprintf(someWriter, "myVar3: %#v -- myVar4: %#+v", myVar3, myVar4)

spew.Sdump()

type ConfigState struct {
	Indent string
	MaxDepth int
	DisableMethods bool
	DisablePointerMethods bool
	DisablePointerAddresses bool
	DisableCapacities bool
	ContinueOnMethod bool
	SortKeys bool
	SpewKeys bool
}
```