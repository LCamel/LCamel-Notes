# Double and Add / Square and Multiply / Montgomery Ladder

## Double and Add

如果有一串 binary digits "1 0 0 1 0", 我們想轉成數字 18 要怎麼做?

其中一種作法是: 從左往右掃, 每次 double 再加進新的位數.

```
1 0 0 1 0
     ^
    n=4
```
我們有個暫存變數 n 裝著目前左邊已處理的數字, 像上面的 4 就是 "1 0 0" 的處理結果.<br>
n = (4 + 4) + 1 = 9 就會是 "1 0 0 1".<br>
n = (9 + 9) + 0 = 18 就會是 "1 0 0 1 0".

## Square and Multiply

如果我們想算 x 的 18 次方要怎麼算?

18 的二進位表示是 "1 0 0 1 0".

其中一種作法是: 從左往右掃, 每次 square 再乘進新的位數.

```
1 0 0 1 0
     ^
    n=x^4
```
我們有個暫存變數 n 裝著目前左邊已處理的數字, 像上面的 x^4 就是 "1 0 0" 的處理結果.<br>
n = (x^4 * x^4) * x^1 = x^9 就會是 "1 0 0 1".<br>
n = (x^9 + x^9) * x^0 = x^18 就會是 "1 0 0 1 0".

## Montgomery Ladder

如果用在處理 elliptic curve 的 [point multiplication](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication),
則前面的方式在處理 bit 0 和 bit 1 時執行的指令和所需的時間可能不同. 有機會從 side channel 洩漏資訊.

Montgomery Ladder 處理了這個問題.

相較於前面只用了一個暫時變數 n, 這邊像 Fibonacci sequence 的程式一樣用了兩個先前的變數.<br>
一個 n0 是原來的 n.<br>
一個 n1 是 n0 + 1.

```
1 0 0 1 0
     ^
    n0=4  n1=n0+1=5
```

我們盡量把新的 new_n0 和 new_n1=new_n0+1 只用 n0 n1 表達, 並且盡量用相似的程式碼.

如果遇到 0, 則
```
new_n0 = n0 + n0
new_n1 = (new_n0 + 1) = n0 + n0 + 1 = n0 + n1
```

如果遇到 1, 則
```
new_n0 = n0 + n0 + 1 = n0 + n1
new_n1 = (new_n0 + 1) = n0 + n1 + 1 = n1 + n1
```

這樣兩個 branch 都是執行相似的程式碼了.

再調整一下 assign 的順序, 我們可以不用新的暫存變數.

```
if (bit is zero) {
    n1 = n0 + n1  // add
    n0 = n0 + n0  // double
} else {
    n0 = n0 + n1  // add
    n1 = n1 + n1  // double
}
```

```
1 0 0 1 0
     ^
    n0=4  n1=n0+1=5

100 -> 1001
n0 = n0 + n1 = 4 + 5 = 9
n1 = n1 + n1 = 5 + 5 = 10

1001 -> 10010
n1 = n0 + n1 = 9 + 10 = 19
n0 = n0 + n0 = 9 + 9 = 18    <==
```

更詳細的分析可以看[這篇](https://cr.yp.to/bib/2003/joye-ladder.pdf).
