How to see the pseudo-assembly output from the compiler:

```bash
go build -gcflags '-S' src/variable_access/locals_vs_pointers.go | less
```

How to see the actual generated assembly:

```bash
go build -gcflags '-l' src/variable_access/locals_vs_pointers.go
go tool objdump bin/variable_access | grep locals_vs_pointers.go | less
```

Using objdump to disassemble:

```bash
objdump -d bin/variable_access | less
```

### References:

 - [Plan 9 assembler manual](http://plan9.bell-labs.com/sys/doc/asm.html)
 - [Plan 9 compiler manual](http://plan9.bell-labs.com/sys/doc/comp.html)


# Results of investigating binaries

## Methods that use pointer receivers are cheaper

```java
type Vector2 struct {
    X, Y float64
}

func main() {
    vec := Vector2{2.4, 3.7}
    _ = vec.ManhattanLength()
    _ = vec.ManhattanLengthPointer()
    _ = vec.X + vec.Y
}

func (v Vector2) ManhattanLength() float64 {
    return v.X + v.Y
}

func (v *Vector2) ManhattanLengthPointer() float64 {
    return v.X + v.Y
}
```

Even though `ManhattanLength()` is inlined, it still requires copying `vec1` onto the top of the stack, even though `vec1` is already on the stack! For a small struct, copying requires several `MOV` instructions. For a larger struct (at least 4 ints or floats), it would require a call to `runtime.duffcopy`. Calling `runtime.duffcopy` takes time and prevents reordering of nearby instructions to minimize stalls.

Let's take a look at the generated assembly instructions:

```java
// Inlined copy of ManhattanLength()
// 6 instructions, 49 bytes
f20f10ac2408010000              REPNE MOVSD_XMM 0x108(SP), X5
f20f10942410010000              REPNE MOVSD_XMM 0x110(SP), X2
f20f100425a0ba4e00              REPNE MOVSD_XMM $f64.0000000000000000(SB), X0
f20f11ac2418010000              REPNE MOVSD_XMM X5, 0x118(SP)
f20f11942420010000              REPNE MOVSD_XMM X2, 0x120(SP)
f20f58ea                        REPNE ADDSD X2, X5

// Inlined copy of ManhattanLengthPointer()
// 5 instructions, 30 bytes
488d9c24f8000000                LEAQ 0xf8(SP), BX
f20f100425a0ba4e00              REPNE MOVSD_XMM $f64.0000000000000000(SB), X0
f20f1023                        REPNE MOVSD_XMM 0(BX), X4
f20f104b08                      REPNE MOVSD_XMM 0x8(BX), X1
f20f58e1                        REPNE ADDSD X1, X4
```

Curiously, two of the instructions in the inlined `ManhattanLength()` are entirely unneeded: `MOVSD_XMM X5, 0x118(SP)` and `MOVSD_XMM X2, 0x120(SP)`. These make a copy of `vec1` on the stack, but that copy is never used since the data is already available in registers `X2` and `X5`.

### Profiling results

```python
func main() {
	vectors := [2]Vector3{ {1.1, 2.2, 9.9}, {1.6, 2.8, 1.3} }
	result := 0.0
	var i1, i2 int

	for i := 0; i < 1000000000; i++ {
		i1 = i & 1
		i2 = ^i & 1
		// Passing by value
		result += Vector3DotVector3(vectors[i1], vectors[i2])
		// Passing by pointer
		//result += Vector3DotVector3Pointer(&vectors[i1], &vectors[i2])
	}

	fmt.Println(result)
}

func Vector3DotVector3(v1, v2 Vector3) float64 {
	return v1.X*v2.X + v1.Y*v2.Y + v1.Z*v2.Z
}

func Vector3DotVector3Pointer(v1, v2 *Vector3) float64 {
	return v1.X*v2.X + v1.Y*v2.Y + v1.Z*v2.Z
}
```

Operation             | Passing by value | Passing by pointer | Manually inlined
----------------------|-----------------:|-------------------:|-----------------:
Vector dot product    | 4.32s            | 2.97s              | -
Vector length squared | 2.60s            | 2.22s              | 2.20s
Vector length         | 10.4s            | 8.43s              | -


### References

- [Profiling go programs](http://blog.golang.org/profiling-go-programs)