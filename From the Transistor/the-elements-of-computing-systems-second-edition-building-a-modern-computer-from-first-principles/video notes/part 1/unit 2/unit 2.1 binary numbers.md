# representing numbers
binary->decimal

# binary addition

half adder - adds two bits

| a   | b   | sum | carry |
| --- | --- | --- | ----- |
| 0   | 0   | 0   | 0     |
| 0   | 1   | 1   | 0     |
| 1   | 0   | 1   | 0     |
| 1   | 1   | 0   | 1     |



full adder - adds three bits

| a   | b   | c   | sum | carry |
| --- | --- | --- | --- | ----- |
| 0   | 0   | 0   | 0   | 0     |




adder - adds two numbers


# negative numbers
1000 -0 
1001 -1
这种方式有些问题：
- -0
- 实现上需要处理不同的情况

使用2's complement
represent negative number -x using the positive number:
2^n-x

0110 6
0111 7
1000 -8 8
1001 -7 9
1010 -6 10

可以观察到上面，负数是 2^n - 1 得到
positive numbers in the range: 0...2^(n-1) - 1
negative numbers in the range: -1...-2^(n-1)


# computing -x

![[Pasted image 20231030182159.png]]
写成2^n - 1 是为了计算方便，变成了全1

Input: 4

Output: -4

-4: In 2's complement 12(ten) -- 1100

4：0100 
-4： 1111 - 0100 + 0001 == 1100



