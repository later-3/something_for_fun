Boolean functions can be realized using logic gates

# Design: from requirements to interface
实现xor

| a   | b   | out |
| --- | --- | --- |
| 0   | 0   | 0   |
| 0   | 1   | 1   |
| 1   | 0   | 1   |
| 1   | 1   | 0    |

Requirement: Build a gate that delivers this functionality

```hdl
CHIP Xor {
    IN a, b;
    OUT out;

    PARTS:
    // Put your code here:
    Nand(a=a, b=b, out=n);
    Nand(a=a, b=n, out=A);
    Nand(a=n, b=b, out=B);
    Nand(a=A, b=B, out=out);
}
```

PARTS: describe how this chip is actually designed.

Gate interface: implemented as an HDL stub file

# General idea:
out = 1 when:
a And Not(b)
Or
b And Not(b)

在完成hdl的时候，对着等于1的条件完成

# Design: from gate diagram to HDL

# HDL: some comments

