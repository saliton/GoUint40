[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/saliton/GoUint40/blob/main/GoUint40.ipynb)

# Go言語で整数を5バイトに押し込む簡単なお仕事

## はじめに

ビッグデータを処理するような場合、データの読み込みが律速になることが多いです。その場合、データのストレージ上のサイズが小さくなると読み込み時間が減り、全体の処理時間が半分になったりします。そんな時、整数が40bitで十分ならば、40bitで保持したいですね。というわけで、Go言語で整数を40bitでストレージに読み書きする方法について調べました。

> ※ インデックス算出の際に時間計測に含めるべきではない演算を行っていたので、それを排除した内容に差し替えました。

まずは、Go言語をインストールします。


```shell
!wget https://golang.org/dl/go1.17.2.linux-amd64.tar.gz
!tar -C /usr/local -xzf go1.17.2.linux-amd64.tar.gz

import os
os.environ['PATH'] += ":/usr/local/go/bin"
```

    --2022-01-20 02:05:24--  https://golang.org/dl/go1.17.2.linux-amd64.tar.gz
    Resolving golang.org (golang.org)... 172.217.204.141, 2607:f8b0:400c:c15::8d
    Connecting to golang.org (golang.org)|172.217.204.141|:443... connected.
    HTTP request sent, awaiting response... 301 Moved Permanently
    Location: https://go.dev/dl/go1.17.2.linux-amd64.tar.gz [following]
    --2022-01-20 02:05:24--  https://go.dev/dl/go1.17.2.linux-amd64.tar.gz
    Resolving go.dev (go.dev)... 216.239.32.21, 216.239.36.21, 216.239.38.21, ...
    Connecting to go.dev (go.dev)|216.239.32.21|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://dl.google.com/go/go1.17.2.linux-amd64.tar.gz [following]
    --2022-01-20 02:05:25--  https://dl.google.com/go/go1.17.2.linux-amd64.tar.gz
    Resolving dl.google.com (dl.google.com)... 74.125.31.190, 74.125.31.91, 74.125.31.93, ...
    Connecting to dl.google.com (dl.google.com)|74.125.31.190|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 134803982 (129M) [application/x-gzip]
    Saving to: ‘go1.17.2.linux-amd64.tar.gz’
    
    go1.17.2.linux-amd6 100%[===================>] 128.56M   230MB/s    in 0.6s    
    
    2022-01-20 02:05:25 (230 MB/s) - ‘go1.17.2.linux-amd64.tar.gz’ saved [134803982/134803982]
    



```shell
!go version
```

    go version go1.17.2 linux/amd64


## 64bit整数の8バイト配列化

> unsafe.Pointerが最速


まずは64bitの整数をバイト配列にする方法を調べます。(以下では整数のエンコーディングはリトルエンディアンにします。特にunsafe.PointerがらみではビッグエンディアンのCPUでは間違った動作になりますのでご注意ください。)

愚直だとこうでしょうか。
```
for j := 0; j < 8; j += 1 {
    buf[j] = byte((v >> (8 * j)) & 0xFF)
}
```


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }
	buf := make([][8]byte, size)

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			v := vs[idx]
			b := &buf[idx]
			for j := 0; j < 8; j += 1 {
				b[j] = byte(v >> (8 * j))
			}
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了

 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 8; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total) 
}
```

    Writing measure.go



```shell
!go run measure.go
```

    2m18.267307953s
    [82 253 252 7 33 0 0 0]
    63808521


流石にfor文で1バイトずつ処理していたら遅いです。

ちなみに、時間計測外の処理は、最適化による計測対象コードの削除防止と、処理の正しさの確認のために付与しています。


では定石通り、内側のfor文を展開しましょう。


```Go
%%writefile measure.go
package main

import (
	"fmt"
    "math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }
	buf := make([][8]byte, size)

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
			v := vs[idx]
			b := &buf[idx]
			b[0] = byte(v)
			b[1] = byte(v >> 8)
			b[2] = byte(v >> 16)
			b[3] = byte(v >> 24)
			b[4] = byte(v >> 32)
			b[5] = byte(v >> 40)
			b[6] = byte(v >> 48)
			b[7] = byte(v >> 56)
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了

 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 8; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total) 
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    39.898105281s
    [82 253 252 7 33 0 0 0]
    63808521


かなり高速化されました。まあ、for文で1バイトずつが遅すぎましたね。

次にライブラリを使いましょう。encoding/binaryライブラリにはエンコーディングを指定した上で整数をバイト配列に書き込む関数が用意されています。


```Go
%%writefile measure.go
package main

import (
	"encoding/binary"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }
	buf := make([][]byte, 0, size)
	for i := 0; i < size; i += 1 {
		buf = append(buf, make([]byte, 8))
    }

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			binary.LittleEndian.PutUint64(buf[idx], vs[idx])
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了

 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 8; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total) 
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    13.314782348s
    [82 253 252 7 33 0 0 0]
    63808521


速くなりました。

ここで禁断のunsafeが登場です。高速化のためにはunsafeは避けて通れません。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"time"
	"math/rand"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size][8]byte

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
	        *(*uint64)(unsafe.Pointer(&buf[idx][0])) = vs[idx]
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 8; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    7.997059849s
    [82 253 252 7 33 0 0 0]
    63808521


さらに速くなりました。

## 8バイト配列から64bit整数への変換

> binary.LittleEndian.Uint64()がお勧め

8バイトの配列をリトルエンディアンとして64bitの整数に変換する素朴な実装は以下になります。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	buf := make([][]byte, size)
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
 	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            b := buf[idx]
            v := uint64(b[0])
            for j := 1; j < 8; j += 1 {
                v += uint64(b[j]) << (8 * j)
            }
            total += v
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    2m52.441620364s
    14225278786399272160


次に内側のfor文を展開したもの。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	buf := make([][]byte, size)
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
 	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            b := buf[idx]
            v := uint64(b[0])
            v += uint64(b[1]) << 8
            v += uint64(b[2]) << 16
            v += uint64(b[3]) << 24
            v += uint64(b[4]) << 32
            v += uint64(b[5]) << 40
            v += uint64(b[6]) << 48
            v += uint64(b[7]) << 56
            total += v
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    39.030929653s
    14225278786399272160


それではライブラリを使ってみましょう。


```Go
%%writefile measure.go
package main

import (
	"encoding/binary"
	"fmt"
    "math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	buf := make([][]byte, size)
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
    		total += binary.LittleEndian.Uint64(buf[idx])
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了

    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    12.682570622s
    14225278786399272160


相当速くなりました。

次は`unsafe.Pointer()`による実装。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	buf := make([][]byte, size)
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
    		total += *(*uint64)(unsafe.Pointer(&buf[idx][0]))
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了

    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    12.853978123s
    14225278786399272160


ほぼ同じです。ただ、CPUがリトルエンディアンである必要があることに注意してください。

## 40bit整数の5バイト配列化

> for文を展開するのがおすすめ

次に40bit(5バイト)を考えましょう。64bitのデータから40bitを取り出す処理が必要です。

まずは素朴な実装。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }

	var buf [size][5]byte

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			v := vs[idx]
			b := &buf[idx]
            for j := 0; j < 5; j += 1 {
    			b[j] = byte(v >> (8 * j))
            }
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 5; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    1m30.425536975s
    [82 253 252 7 33]
    63808521


ちょうど8回のループが5回になった分だけ処理時間が短くなっています。



次に内側のfor文を展開したもの。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }

	var buf [size][5]byte

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			v := vs[idx]
			b := &buf[idx]
			b[0] = byte(v)
			b[1] = byte(v >> 8)
			b[2] = byte(v >> 16)
			b[3] = byte(v >> 24)
			b[4] = byte(v >> 32)
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 5; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    10.798947945s
    [82 253 252 7 33]
    63808521


処理時間は64bit整数の時の8分の5よりだいぶ短いです。原因はよくわかりませんが、格納するバッファの大きさが小さくなったのが影響している可能性があります。

次はライブラリ使用です。`PutUint64()`は引数に８バイト以上のスライスしかとれないので、`copy(dst, src)`を使って8バイトに書き込んだworkから５バイト分コピーします。


```Go
%%writefile measure.go
package main

import (
	"encoding/binary"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	var buf [size][]byte
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
		buf[i] = make([]byte, 5)
    }

	work := make([]byte, 8)

	// 計測開始
	start := time.Now()
	for i := 0; i < count / size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			binary.LittleEndian.PutUint64(work, vs[idx])
			copy(buf[idx], work[:5])
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 5; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total)
 }
```

    Overwriting measure.go



```shell
!go run measure.go
```

    47.237593761s
    [82 253 252 7 33]
    63808521


遅くなりました。やはり複製は重たい。

unsafe.Pointer()ではどうでしょうか。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"time"
	"math/rand"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	var buf [size][]byte
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
		buf[i] = make([]byte, 5)
    }

	var work [8]byte
 
	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			*(*uint64)(unsafe.Pointer(&work[0])) = vs[idx]
			copy(buf[idx], work[:5])
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
 	// 時間計測外の処理
    fmt.Println(buf[0])
	total := 0
	for i := 0; i < size; i += 1 {
		for j := 0; j < 5; j += 1 {
			  total += int(buf[i][j])
		}
	}
	fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    48.494321196s
    [82 253 252 7 33]
    63808521


ライブラリ使用と同じです。複製の処理が律速になっているようです。

複製はせずに、読み込む時に5バイトだけ取り出すようにしたらどうでしょうか。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"time"
	"math/rand"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }

	var buf [8]byte
	total := uint64(0)

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			*(*uint64)(unsafe.Pointer(&buf[0])) = vs[idx]
	        total += *(*uint64)(unsafe.Pointer(&buf[0])) & 0xFF_FFFF_FFFF
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
    fmt.Println(buf, total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    12.294887158s
    [25 237 192 70 183 0 0 0] 5692317074739218432


だいぶ速くなりましたが、for文の展開には及びません。

0xFF_FFFF_FFFFのマスクを工夫してみます。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"time"
	"math/rand"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var vs [size]uint64
	for i := 0; i < size; i += 1 {
		vs[i] = uint64(rand.Intn(1<<40))
    }

	var buf [8]byte
	total := uint64(0)

	// 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			*(*uint64)(unsafe.Pointer(&buf[0])) = vs[idx]
	        total += uint64(*(*uint32)(unsafe.Pointer(&buf[0]))) + uint64(buf[4]) << 32
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
    fmt.Println(buf, total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    12.246566138s
    [25 237 192 70 183 0 0 0] 5692317074739218432


ほぼ同じです。

## 5バイト配列から40bit整数への変換

> `v = uint64(binary.LittleEndian.Uint32(buf)) + uint64(buf[4]) << 32`がおすすめ

5バイトの配列を40bitの整数として読み出します。まずは素朴な実装から。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var buf [size][]byte
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
 	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            b := buf[idx]
            v := uint64(0)
            for j := 0; j < 5; j += 1 {
                v += uint64(b[j]) << (8 * j)
            }
            total += v
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    1m39.081932045s
    14634206628677970048


素朴な実装では整数から配列に分配するのと同等の時間がかかっています。

次に内側のfor文の展開


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var buf [size][]byte
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

    // 計測開始
 	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            b := buf[idx]
            v := uint64(b[0])
            v += uint64(b[1]) << 8
            v += uint64(b[2]) << 16
            v += uint64(b[3]) << 24
            v += uint64(b[4]) << 32
            total += v
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    23.83489207s
    14634206628677970048


この場合は、配列から整数に戻す方が時間がかかるようです。

ライブラリ使用ではどうでしょうか。


```Go
%%writefile measure.go
package main

import (
	"encoding/binary"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var buf [size][]byte
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)), 0, 0, 0}
    }

	// 計測開始
	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			total += binary.LittleEndian.Uint64(buf[idx]) & 0xFF_FFFF_FFFF
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    13.522700543s
    14634206628677970048


改善しました。ただ、ライブラリを利用するために、配列を５バイトではなく8バイトに水増しするズルをしています。

0xFF_FFFF_FFFFでマスクする処理を変更してみましょう。配列の前半4バイトを32bitの整数として、5バイト目を32ビットずらして加算することでマスク処理を回避します。


```Go
%%writefile measure.go
package main

import (
	"encoding/binary"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var buf [size][]byte
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256))}
    }

	// 計測開始
	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			b := buf[idx]
			total += uint64(binary.LittleEndian.Uint32(b)) + (uint64(b[4]) << 32)
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    14.439393482s
    14634206628677970048


少し遅くなりました。ただ、こちらは8バイトへの水増しが不要です。

`unsafe.Pointer()`を使った場合はどうでしょうか。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
	var buf [size][]byte
    for i := 0; i < size; i += 1 {
        buf[i] = []byte{byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)),
                        byte(rand.Intn(256)), 0, 0, 0}
    }

	// 計測開始
 	start := time.Now()
 	total := uint64(0)
	for i := 0; i < count/size; i += 1 {
		for idx := 0; idx < size; idx += 1 {
			total += *(*uint64)(unsafe.Pointer(&buf[idx][0])) & 0xFF_FFFF_FFFF
		}
	}
	fmt.Println(time.Since(start))
	// 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    13.480186306s
    14634206628677970048


ライブラリと同程度です。CPUがリトルエンディアンである必要もあることですし、わざわざu`nsafe.Pointer()`を使う意味はないようです。

## 40bit整数の配列の5Nバイト配列化

> `*(*uint64)(unsafe.Pointer(&buf[j])) = v`がおすすめ

次に整数の配列をバイト配列化する場合を測定しましょう。10万個の整数を5バイトずつのバイト配列にします。

1バイトずつ書き写す素朴な処理は以下になります。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5]byte
 
    // 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            v := ans[idx]
            b := buf[(idx*5):(idx*5+5)]
            for j := 0; j < 5; j += 1 {
                b[j] = byte(v >> (8 * j))
            }
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    total := uint64(0)
    for i := 0; i < size * 5; i += 1 {
        total += uint64(buf[i])
    }
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    1m43.655985153s
    63808521


内側のfor文を展開した場合


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5]byte
 
    // 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            v := ans[idx]
            b := buf[(idx*5):(idx*5+5)]
            b[0] = byte(v)
            b[1] = byte(v >> 8)
            b[2] = byte(v >> 16)
            b[3] = byte(v >> 24)
            b[4] = byte(v >> 32)
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    total := uint64(0)
    for i := 0; i < size * 5; i += 1 {
        total += uint64(buf[i])
    }
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    15.917946557s
    63808521


Goの場合は、for文の展開が効きますね。この最適化は自動的にはなされないようです。

Go言語には上記のように１バイトずつ書き込むほかには、４バイトずつ、あるいは、８バイトずつ書き込むモジュールがあります。`binary.LittleEndian.PutUint32()`と`binary.LittleEndian.PutUint64()`です。これらはCPUがネイティブで複製命令を持つサイズであるため、高速に実行されます。しかし、5バイトを書き込むものはありません。そこでリトルエンディアンの並びを利用して、以下のように5バイトずらしながら8バイトずつ書き込むことで所望の結果を得ます。

![bytes.png](bytes.png)

以下がそのコードです。


```Go
%%writefile measure.go
package main

import (
    "encoding/binary"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5 + 3]byte
 
    // 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            binary.LittleEndian.PutUint64(buf[idx*5:], ans[idx])
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    total := uint64(0)
    for i := 0; i < size * 5; i += 1 {
        total += uint64(buf[i])
    }
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    15.710748546s
    63808521


for文の展開とほぼ同じ速度です。

次は`unsafe.Pointer()`で実装します。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5 + 3]byte

    // 計測開始
	start := time.Now()
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size; idx += 1 {
            *(*uint64)(unsafe.Pointer(&buf[idx*5])) = ans[idx]
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    total := uint64(0)
    for i := 0; i < size * 5; i += 1 {
        total += uint64(buf[i])
    }
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    11.727558834s
    63808521


いよいよ本領を発揮しました。速いです。

## 5Nバイト配列から40bit整数の配列への変換

> `v = uint64(*(*uint32)(unsafe.Pointer(&buf[j]))) + uint64(buf[j+4]) << 32`がおすすめ

40bit整数が並んだバイト配列から整数を読み出す場合を測定しましょう。5バイトずつのバイト配列から10万個の整数を読み出します。素朴な実装は以下になります。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5]byte
 
    for j := 0; j < size; j += 1 {
        v := ans[j]
        *(*uint64)(unsafe.Pointer(&buf[j*5])) = v
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
    for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size * 5; idx += 5 {
            b := buf[idx:idx+5]
            v := uint64(b[0])
            v += uint64(b[1]) << 8
            v += uint64(b[2]) << 16
            v += uint64(b[3]) << 24
            v += uint64(b[4]) << 32
            total += v
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    23.48296862s
    5692317074739218432


これが基準になります。

次にライブラリ使用で、5バイトのマスク。


```Go
%%writefile measure.go
package main

import (
    "encoding/binary"
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5 + 3]byte
 
    for j := 0; j < size; j += 1 {
        v := ans[j]
        *(*uint64)(unsafe.Pointer(&buf[j*5])) = v
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
    for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size * 5; idx += 5 {
            total += binary.LittleEndian.Uint64(buf[idx:]) & 0xFF_FFFF_FFFF
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了
 
    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    19.391102667s
    5692317074739218432


少し改善。ただ、bufの領域を3バイト増やしています。

次はマスク外し。


```Go
%%writefile measure.go
package main

import (
    "encoding/binary"
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5]byte
 
    for j := 0; j < size; j += 1 {
        v := ans[j]
        *(*uint64)(unsafe.Pointer(&buf[j*5])) = v
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size * 5; idx += 5 {
            total += uint64(binary.LittleEndian.Uint32(buf[idx:])) + uint64(buf[idx+4]) << 32
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了

    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    19.469417185s
    5692317074739218432


ほぼ同じ。ただ、bufの領域はsize*5です。

次は`unsafe.Pointer()`かつ5バイトマスク。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5 + 3]byte
 
    for j := 0; j < size; j += 1 {
        v := ans[j]
        *(*uint64)(unsafe.Pointer(&buf[j*5])) = v
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size * 5; idx += 5 {
            total += *(*uint64)(unsafe.Pointer(&buf[idx])) & 0xFF_FFFF_FFFF
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了

    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    11.761672529s
    5692317074739218432


改善。ただ、bufは3バイト追加しています。

最後にマスク外し。


```Go
%%writefile measure.go
package main

import (
	"fmt"
	"math/rand"
	"time"
    "unsafe"
)

func main() {
	const count = 10_000_000_000
    const size = 100_000
    var ans [size]uint64
    for i := 0; i < size; i += 1 {
    	ans[i] = uint64(rand.Intn(1<<40))
    }
	var buf [size * 5]byte
 
    for j := 0; j < size; j += 1 {
        v := ans[j]
        *(*uint64)(unsafe.Pointer(&buf[j*5])) = v
    }

    // 計測開始
	start := time.Now()
    total := uint64(0)
	for i := 0; i < count/size; i += 1 {
        for idx := 0; idx < size * 5; idx += 5 {
            total += uint64(*(*uint32)(unsafe.Pointer(&buf[idx]))) + uint64(buf[idx+4]) << 32
        }
	}
	fmt.Println(time.Since(start))
    // 計測終了

    fmt.Println(total)
}
```

    Overwriting measure.go



```shell
!go run measure.go
```

    11.717058394s
    5692317074739218432


速度はほぼ同じ。ただ、bufのサイズはsize*5です。

結局、大きな配列に連続して40bit整数を書き込んだり読み込んだりするのには`unsafe.Pointer()`が有効でした。
