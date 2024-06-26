---
title: "[筆記] Go Concurrency Pattern"
categories:
  - 筆記
tags:
  - Golang
toc: true
toc_label: Index
---


concurrency pattern 整理

## generator

返回一個可以持續產生數值的 channel

```golang
package main

import (
	"fmt"
)

func Generator() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 0; i < 10; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func main() {

	ch := Generator()

	for val := range ch {
		fmt.Println(val)
	}

	fmt.Println("done")

}
```

```
0
1
2
3
4
5
6
7
8
9
done

Process finished with the exit code 0
```

## channel done

類似 generator，產生一個可持續輸出的channel, 但可以在從外部控制何時結束

```golang

package main

import (
	"context"
	"fmt"
	"time"
)

func DataGenerator(ctx context.Context) <-chan int {

	ch := make(chan int, 0)

	go func() {
		i := 0
		defer close(ch)

		for {

			select {

			case <-ctx.Done():
				fmt.Println("finish...")
				return
			case ch <- i:
				time.Sleep(time.Millisecond * 100)
				i += 1
			}

		}

	}()

	return ch

}

func main() {

	ctx, cancel := context.WithCancel(context.Background())

	ch := DataGenerator(ctx)

	for val := range ch {

		fmt.Println(val)

		if val == 10 {
			cancel()
			break
		}
	}

	fmt.Println("done")

}

```

```
0
1
2
3
4
5
6
7
8
9
10
done

```

## pipeline

簡單說就是將多個channel串接在一起，每個channel都有自己的處理程序, 之後從最終端channel取得結果

```mermaid
graph LR
    A[DataChannel] --> B[ProcesssChannel1]
    B --> C[ProcessChannel2]
    C --> D[ProcessChannelN...]
```

以下範例

```mermaid
graph LR
    A[DataChannel] --> B[Add 10]
    B --> C[Add 20]
```

```golang
package main

import (
	"context"
	"fmt"
)

func Generator(ctx context.Context) <-chan int {

	ch := make(chan int, 0)

	go func() {
		defer close(ch)
		i := 0
		for {
			select {
			case <-ctx.Done():
				fmt.Println("finish...")
			case ch <- i:
				i += 1
			}
		}
	}()

	return ch

}

func ADD(ctx context.Context, inputChan <-chan int, AddValue int) <-chan int {

	resChan := make(chan int, 0)

	go func() {
		defer close(resChan)

		for {
			select {
			case <-ctx.Done():
				fmt.Println("finish... Add...")
				return
			case val, ok := <-inputChan:
				if !ok {
					return
				}
				resChan <- AddValue + val
			}
		}
	}()

	return resChan
}

func main() {

	ctx, cancel := context.WithCancel(context.Background())

	ch := Generator(ctx)
	ch2 := ADD(ctx, ch, 10)
	ch3 := ADD(ctx, ch2, 20)

	for val := range ch3 {
		fmt.Println(val)
		if val == 35 {
			cancel()
			break
		}
	}

	fmt.Println("done")

}
```

```
30
31
32
33
34
35
done
```

## fan in

將多個channel的輸出合併成一個channel

```mermaid

graph LR
    A[DataChannel1] --> D[FanInChannel]
    B[DataChannel2] --> D
    C[DataChannelN...] --> D
```

範例

```mermaid
graph LR
    A["DataChannel1 (0-4) "] --> D["FanInChannel (0,1,2,3,4,100,101,102,103,104)"]
    B["DataChannel2 (100-104)"] --> D
```

```golang
package main

import "sync"

func Generator1() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 0; i < 5; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func Generator2() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 100; i < 105; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func FanIn(wg *sync.WaitGroup, chs ...<-chan int) chan int {

	out := make(chan int, 0)
	go func() {

		defer close(out)

		for _, ch := range chs {
			wg.Add(1)

			go func(ch <-chan int) {
				for val := range ch {
					out <- val
				}
				wg.Done()
			}(ch)

		}

		wg.Wait()

	}()

	return out

}

func main() {

	wg := sync.WaitGroup{}

	ch1 := Generator1()
	ch2 := Generator2()

	out := FanIn(&wg, ch1, ch2)

	for val := range out {
		println(val)
	}

	println("done")
}
```

```
100
0
1
2
3
4
101
102
103
104
done
```

## fan out

將一個channel的資料分發到多個channel

```mermaid
graph LR
    A[DataChannel] --> B[FanChannel1]
    A --> C[FanOutChannel2]
    A --> D[FanOutChannelN...]
```

範例

實際上分配不一定會平均, 有一定數量時, 分配會接近

```mermaid
graph LR
    A["DataChannel (0-59)"] --> B["FanOutChannel1 (0,3,6...)"]
    A --> C["FanOutChannel2 (1,4,7...)"]
    A --> D["FanOutChannel3 (2,5,8....)"]
```

```golang
package main

import (
	"fmt"
	"sync"
)

func Generator() <-chan string {

	ch := make(chan string, 0)

	go func() {

		for i := 0; i < 60; i++ {
			ch <- fmt.Sprintf("data: %d", i)
		}
		close(ch)
	}()

	return ch
}

func FanOut(SourceChan <-chan string, Name string) <-chan string {

	splitchan := make(chan string, 0)

	go func() {

		for val := range SourceChan {
			splitchan <- fmt.Sprintf("%v assgin to %v", val, Name)
		}
		close(splitchan)

	}()

	return splitchan

}

func main() {

	ch := Generator()
	wp := &sync.WaitGroup{}
	splitchan1 := FanOut(ch, "fan1")
	splitchan2 := FanOut(ch, "fan2")
	splitchan3 := FanOut(ch, "fan3")
	static_num := make(chan int, 3)

	wp.Add(3)
	go func() {
		count := 0
		for val := range splitchan1 {
			fmt.Println(val)
			count += 1
		}
		static_num <- count
		wp.Done()

	}()

	go func() {
		count := 0
		for val := range splitchan2 {
			fmt.Println(val)
			count += 1
		}
		static_num <- count
		wp.Done()
	}()

	go func() {
		count := 0
		for val := range splitchan3 {
			fmt.Println(val)
			count += 1
		}
		static_num <- count
		wp.Done()
	}()

	wp.Wait()
	close(static_num)
	fmt.Println("fan size")
	for val := range static_num {
		fmt.Println(val)
	}
}
```

```golang
data: 2 assgin to fan2
data: 3 assgin to fan2
data: 4 assgin to fan2
data: 5 assgin to fan2
data: 6 assgin to fan2
data: 7 assgin to fan2
data: 0 assgin to fan3
data: 1 assgin to fan1
data: 10 assgin to fan1
data: 11 assgin to fan1
data: 14 assgin to fan1
data: 9 assgin to fan3
data: 13 assgin to fan3
data: 16 assgin to fan3
data: 17 assgin to fan3
data: 19 assgin to fan3
data: 15 assgin to fan1
data: 18 assgin to fan1
data: 8 assgin to fan2
data: 12 assgin to fan2
data: 24 assgin to fan2
data: 21 assgin to fan1
data: 22 assgin to fan1
data: 20 assgin to fan3
data: 27 assgin to fan1
data: 23 assgin to fan3
data: 28 assgin to fan1
data: 29 assgin to fan1
data: 30 assgin to fan1
data: 25 assgin to fan2
data: 26 assgin to fan2
data: 33 assgin to fan2
data: 34 assgin to fan2
data: 37 assgin to fan2
data: 31 assgin to fan1
data: 35 assgin to fan1
data: 32 assgin to fan3
data: 36 assgin to fan3
data: 40 assgin to fan3
data: 41 assgin to fan3
data: 44 assgin to fan3
data: 38 assgin to fan2
data: 42 assgin to fan2
data: 39 assgin to fan1
data: 43 assgin to fan1
data: 46 assgin to fan2
data: 47 assgin to fan1
data: 45 assgin to fan3
data: 51 assgin to fan3
data: 52 assgin to fan3
data: 49 assgin to fan1
data: 54 assgin to fan1
data: 53 assgin to fan3
data: 55 assgin to fan3
data: 56 assgin to fan1
data: 57 assgin to fan1
data: 48 assgin to fan2
data: 50 assgin to fan2
data: 59 assgin to fan1
data: 58 assgin to fan3
fan size
19
22
19
```

## fan in out

簡單說就是結合fan in , fan out, 將多個channel的資料匯集近一個channel, 之後再將資料分配至多個channel  


```mermaid
    graph LR
    A[DataChannel1] --> D[FanInChannel]
    B[DataChannel2] --> D
    C[DataChannelN...] --> D
    D --> E[FanOutChannel1]
    D --> F[FanOutChannel2]
    D --> G[FanOutChannelN...]

```

```golang
package main

import (
	"fmt"
	"sync"
)

func Generator1() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 0; i < 20; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func Generator2() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 80; i < 100; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func FanIn(chs ...<-chan int) <-chan string {

	out := make(chan string, 0)

	go func() {
		wp := &sync.WaitGroup{}
		for _, ch := range chs {
			wp.Add(1)
			go func(ch <-chan int) {

				for val := range ch {
					out <- fmt.Sprintf("data: %v", val)
				}
				wp.Done()
			}(ch)
		}
		wp.Wait()
		close(out)
	}()

	return out

}

func FanOut(SourceChan <-chan string, Name string) <-chan string {

	splitchan := make(chan string, 0)

	go func() {

		for val := range SourceChan {
			splitchan <- fmt.Sprintf("%v assgin to %v", val, Name)
		}
		close(splitchan)

	}()

	return splitchan

}

func main() {

	ch1 := Generator1()
	ch2 := Generator2()
	fanin := FanIn(ch1, ch2)

	wp := &sync.WaitGroup{}
	splitchan1 := FanOut(fanin, "fan1")
	splitchan2 := FanOut(fanin, "fan2")
	splitchan3 := FanOut(fanin, "fan3")
	static_num := make(chan int, 3)

	wp.Add(3)
	go func() {
		count := 0
		for val := range splitchan1 {
			fmt.Println(val)
			count += 1
		}
		wp.Done()
		static_num <- count
	}()

	go func() {
		count := 0
		for val := range splitchan2 {
			fmt.Println(val)
			count += 1
		}
		wp.Done()
		static_num <- count
	}()

	go func() {
		count := 0
		for val := range splitchan3 {
			fmt.Println(val)
			count += 1
		}
		wp.Done()
		static_num <- count
	}()

	wp.Wait()
	close(static_num)
	fmt.Println("fan size")
	for val := range static_num {
		fmt.Println(val)
	}

}
```


## or channel

會設計多個 函數, 該函數會回傳一個channel, 平時堵塞, 只有該函數完成後, 會輸入訊號至該channel  

會監控以上函數的channel, 一旦有一個channel有訊號, 會執行後續的動作 (僅執行一次), 後面監控函數完成後 不會再執行

```mermaid
graph LR
    A[ActivateFunctionAfter1] -.->|send| B[or_channel]
    C[ActivateFunctionAfter2] -->|send| B
    D[ActivateFunctionAfter3] -.->|send| B
    B -->|one of them trigger| F[once任務]

```



```golang
package main

import (
	"fmt"
	"sync"
	"time"
)

// 該函數主要是輸出一個channel, 這邊模擬處理任務處理 完成後 close channel, 使其不阻塞
func ActivateFunctionAfter(duration time.Duration) <-chan struct{} {

	c := make(chan struct{})
	time.AfterFunc(duration, func() {
		fmt.Println("任務完成")
		close(c)
	})
	return c
}

// 用於傳入多個channel, 這邊目的是其中之一完成後(不阻塞), 執行一次特定任務
func or_channel(wp *sync.WaitGroup, chans ...<-chan struct{}) <-chan struct{} {

	ch := make(chan struct{}, 0) // 用於傳出 once任務已經被觸發
	once := new(sync.Once)

	go func() {

		for _, inputChan := range chans {
			wp.Add(1)
			go func(inputChan <-chan struct{}) {

				<-inputChan // 等待 activeFunctionAfter 其中之一被完成

				once.Do(func() { // once 任務 只會被觸發一次

					fmt.Println("執行once任務")
					// once 任務 內容 ..... 這邊只是模擬

					ch <- struct{}{} // once 完成後 傳出
					close(ch)
				})
				wp.Done()
			}(inputChan)
		}
	}()

	return ch
}

// 主要用於多個任務 , 只要其中之一完成後, 執行once任務 (只想被觸發一次,後續其他任務不會再被觸發)
func main() {
	wp := &sync.WaitGroup{}
	// 傳入想監控的函數channel, 函數完成後會觸發once任務
	res := or_channel(wp,
		ActivateFunctionAfter(time.Second*5),
		ActivateFunctionAfter(time.Second*2),
		ActivateFunctionAfter(time.Second*3),
		ActivateFunctionAfter(time.Second*4),
	)

	<-res // 監控once任務完成
	wp.Wait()

}
```
```
任務完成
執行once任務
任務完成
任務完成
任務完成
```

## or channel done

與 or channel 類似,  監控多個函數的完成channel, 一旦有一個函數完成後, 會執行後續的動作, 但多了可從外部trigger直接執行後續動作

```mermaid

graph LR
    A[ActivateFunctionAfter1] -.->|send| B[or_channel]
    C[ActivateFunctionAfter2] -->|send| B
    D[ActivateFunctionAfter3] -.->|send| B
    E[context] -.->|trigger| B
    B -->|one of them trigger| F[once任務]

```


```golang
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func ActivateFunctionAfter(duration time.Duration) <-chan struct{} {
	c := make(chan struct{})
	time.AfterFunc(duration, func() {
		fmt.Println("任務完成")
		close(c)
	})
	return c
}

func or_channel_done(ctx context.Context, wp *sync.WaitGroup, chans ...<-chan struct{}) <-chan struct{} {
	ch := make(chan struct{})
	once := new(sync.Once)
	go func() {
		for _, inputChan := range chans {
			go func(inputChan <-chan struct{}) {
				wp.Add(1)
				defer wp.Done()
				for {
					select {
					case <-ctx.Done():
						fmt.Println("任務逾時....")
						once.Do(func() {
							fmt.Println("timeout, 執行once任務")
							close(ch)
						})
						return
					case <-inputChan:
						once.Do(func() {
							fmt.Println("執行once任務")
							ch <- struct{}{}
							close(ch)
						})
						return
					}
				}
			}(inputChan)
		}
	}()
	return ch
}

func main() {
	wp := &sync.WaitGroup{}
	ctx, _ := context.WithTimeout(context.Background(), time.Second*5)
	ch1 := ActivateFunctionAfter(time.Second * 7)
	ch2 := ActivateFunctionAfter(time.Second * 11)
	ch3 := ActivateFunctionAfter(time.Second * 9)
	res := or_channel_done(ctx, wp, ch1, ch2, ch3)
	<-res
	wp.Wait()
```

```
任務逾時....
timeout, 執行once任務
任務逾時....
任務逾時....
```


## tee channel

將一個channel 產生duplicate, 若將資料送入該channel, duplicate的channel 也會收到資料  

實際作法為 原始channel會保持被listen , 只要發現有資料送入, 就會在duplicate的channel也送入相同的資料  

```mermaid
graph LR
    A[Data] -->|input| B[TeeChannel1]
    B -->|tee| C[TeeChannel2]
    B -->|tee| D[TeeChannel3]
    C -->|pull| E[Data]
    D -->|pull| F[Data]
```

在原始channel放入1-4, 從duplicate channel 讀出資料  
```golang

package main

import (
	"fmt"
	"sync"
)

func Generator() <-chan int {

	ch := make(chan int, 0)

	go func() {

		for i := 0; i < 5; i++ {
			ch <- i
		}

		close(ch)
	}()

	return ch

}

func tee(input <-chan int, outputs []chan int) {
	go func() {
		wg := &sync.WaitGroup{}

		for data := range input {
			for _, output := range outputs {
				wg.Add(1)
				go func(data int, output chan int) {
					defer wg.Done()
					output <- data
				}(data, output)
			}
			wg.Wait()
		}

		for _, output := range outputs {
			close(output)
		}

	}()
}

// 簡單說就是把一個channel 產生多個duplicate, 與fan out不同的是, fan out 也是一分多 但資料的總數不會變多 (一包餅乾變成兩個人在吃)
// tee則是真的一包餅乾變兩包
func main() {
	wp := &sync.WaitGroup{}
	ch := Generator()

	outputs := make([]chan int, 2)

	for i := 0; i < len(outputs); i++ { // 初始化 outputs
		outputs[i] = make(chan int, 0)
	}

	tee(ch, outputs)

	for index, output := range outputs {
		wp.Add(1)
		go func(index int, chanInt chan int) {
			for val := range chanInt {
				fmt.Printf("%v chan: %v\n", index, val)
			}
			wp.Done()
		}(index, output)
	}
	wp.Wait()
}
```

```
1 chan: 0
1 chan: 1
0 chan: 0
1 chan: 2
0 chan: 1
1 chan: 3
0 chan: 2
1 chan: 4
0 chan: 3
0 chan: 4
```


## bridge channel

將多個channel的資料合併成一個channel輸出, 可保證順序性 

```mermaid
graph LR
    A[DataChannel1] -->|1,2,3| B[BridgeChannel]
    C[DataChannel2] -->|11,12,13| B
    D[DataChannelN...] -->|101,102,103| B
    B -->|poll| E["[1,2,3] [101,102,103] [11,12,13]"]
```

```golang
package main

import (
	"context"
	"fmt"
)

func Generator(start int, end int) <-chan int {
	ch := make(chan int)

	go func() {
		defer close(ch)
		for i := start; i < end; i++ {
			ch <- i
		}
	}()

	return ch
}

func StreamGenerator(chs ...<-chan int) <-chan (<-chan int) {
	ch := make(chan (<-chan int))

	go func() {

		for _, val := range chs {
			ch <- val
		}

		close(ch)
	}()

	return ch
}

func Bridge(ctx context.Context, ch_ch <-chan <-chan int) <-chan int {
	res := make(chan int)

	go func() {
		defer close(res)
		for {
			select {
			case <-ctx.Done():
				return
			case ch, ok := <-ch_ch:
				if !ok {
					return
				}
				for val := range ch {
					res <- val
				}
			}
		}
	}()

	return res
}

func main() {

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	ch1 := Generator(100, 105)
	ch2 := Generator(6, 10)
	ch3 := Generator(500, 507)

	ch_ch := StreamGenerator(ch1, ch2, ch3)

	for val := range Bridge(ctx, ch_ch) {
		fmt.Println(val)
	}

}
```

```
100
101
102
103
104
6
7
8
9
500
501
502
503
504
505
506
```


## worker pool

建立一個固定數量的worker goroutine, 用於處理任務,  通常會與cpu數量相同 (可同時處理的任務數量)  
基本上就是 producer 與 consumer概念,  producer 會將任務放入channel, consumer 會從channel取出任務處理  

```mermaid
graph LR
    A[Producer] -->|task| B[TaskParmChannel]
    B -->|taskParm1| C[Worker1]
    B -->|taskParm2| D[Worker2]
    B -->|taskParm3| E[Worker3]
    B -->|taskParmN..| F[WorkerN...]
    C -->|result| G[OutputChannel]
    D -->|result| G
    E -->|result| G
    F -->|result| G
```

範例   
這邊假設有五十組任務, 啟動16個worker, 每個任務需花費5秒, 這邊會等待所有任務完成後, 並計算總耗時  
```golang
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func MyTask(WorkerID int, para int) string {

	time.Sleep(time.Second * 5)
	fmt.Println("WorkerID:", WorkerID, "task_id:", para, "執行中")
	return fmt.Sprintf("worker %v 已完成task %v ########", WorkerID, para)

}

func Worker(ctx context.Context, once *sync.Once, workerID int, inputParam <-chan int, output chan string) {

	defer once.Do(func() {
		fmt.Println("Worker:", workerID, "等待最後一組結束")
		time.Sleep(time.Second * 6)
		close(output)
	})

	for {
		select {
		case <-ctx.Done():
			fmt.Println("worker已逾時.........close worker", workerID)
			return
		case p, ok := <-inputParam:
			if !ok || ctx.Err() != nil {
				return
			}
			res := MyTask(workerID, p)
			output <- res
		}
	}
}

func GenerateTaskParm() chan int {
	inputParam := make(chan int)

	go func() {
		defer close(inputParam)
		for i := 0; i < 50; i++ {
			inputParam <- i
			time.Sleep(time.Millisecond * 100)
		}
	}()

	return inputParam
}

func GenerateWorkerID(CPUNum int) chan int {
	workerID := make(chan int)

	go func() {
		for i := 0; i < CPUNum; i++ {
			workerID <- i
		}
		close(workerID)
	}()

	return workerID
}

func main() {

	now := time.Now()

	CPUNum := 16
	// 用來紀錄goroutine worker編號
	workerID := GenerateWorkerID(CPUNum)

	// 函數參數
	para := GenerateTaskParm()

	// 用於返回 task完成後的結果
	output := make(chan string)

	wg := &sync.WaitGroup{}
	wg.Add(CPUNum)
	ctx, _ := context.WithTimeout(context.Background(), time.Second*100)
	once := new(sync.Once)
	go func() {
		for i := 0; i < CPUNum; i++ {
			go func() {
				defer wg.Done()
				WorkerID := <-workerID
				Worker(ctx, once, WorkerID, para, output)
			}()
		}
	}()
	count := 0
	go func() {
		for val := range output {
			fmt.Println(val)
			count += 1
		}
	}()
	wg.Wait()
	cost := time.Since(now)
	fmt.Println("總計完成數量:", count)
	fmt.Println("總計耗時:", cost)

}
```

```
WorkerID: 0 task_id: 0 執行中
worker 0 已完成task 0 ########
WorkerID: 15 task_id: 1 執行中
worker 15 已完成task 1 ########
WorkerID: 1 task_id: 2 執行中
worker 1 已完成task 2 ########
WorkerID: 2 task_id: 3 執行中
worker 2 已完成task 3 ########
WorkerID: 3 task_id: 4 執行中
worker 3 已完成task 4 ########
WorkerID: 4 task_id: 5 執行中
worker 4 已完成task 5 ########
WorkerID: 10 task_id: 6 執行中
worker 10 已完成task 6 ########
WorkerID: 11 task_id: 7 執行中
worker 11 已完成task 7 ########
WorkerID: 12 task_id: 8 執行中
worker 12 已完成task 8 ########
WorkerID: 13 task_id: 9 執行中
worker 13 已完成task 9 ########
WorkerID: 7 task_id: 10 執行中
worker 7 已完成task 10 ########
WorkerID: 9 task_id: 11 執行中
worker 9 已完成task 11 ########
WorkerID: 6 task_id: 12 執行中
worker 6 已完成task 12 ########
WorkerID: 8 task_id: 13 執行中
worker 8 已完成task 13 ########
WorkerID: 5 task_id: 14 執行中
worker 5 已完成task 14 ########
WorkerID: 14 task_id: 15 執行中
worker 14 已完成task 15 ########
WorkerID: 0 task_id: 16 執行中
worker 0 已完成task 16 ########
WorkerID: 15 task_id: 17 執行中
worker 15 已完成task 17 ########
WorkerID: 1 task_id: 18 執行中
worker 1 已完成task 18 ########
WorkerID: 2 task_id: 19 執行中
worker 2 已完成task 19 ########
WorkerID: 3 task_id: 20 執行中
worker 3 已完成task 20 ########
WorkerID: 4 task_id: 21 執行中
worker 4 已完成task 21 ########
WorkerID: 10 task_id: 22 執行中
worker 10 已完成task 22 ########
WorkerID: 11 task_id: 23 執行中
worker 11 已完成task 23 ########
WorkerID: 12 task_id: 24 執行中
worker 12 已完成task 24 ########
WorkerID: 13 task_id: 25 執行中
worker 13 已完成task 25 ########
WorkerID: 7 task_id: 26 執行中
worker 7 已完成task 26 ########
WorkerID: 9 task_id: 27 執行中
worker 9 已完成task 27 ########
WorkerID: 6 task_id: 28 執行中
worker 6 已完成task 28 ########
WorkerID: 8 task_id: 29 執行中
worker 8 已完成task 29 ########
WorkerID: 5 task_id: 30 執行中
worker 5 已完成task 30 ########
WorkerID: 14 task_id: 31 執行中
worker 14 已完成task 31 ########
WorkerID: 0 task_id: 32 執行中
worker 0 已完成task 32 ########
WorkerID: 15 task_id: 33 執行中
worker 15 已完成task 33 ########
WorkerID: 1 task_id: 34 執行中
worker 1 已完成task 34 ########
Worker: 1 等待最後一組結束
WorkerID: 2 task_id: 35 執行中
worker 2 已完成task 35 ########
WorkerID: 3 task_id: 36 執行中
worker 3 已完成task 36 ########
WorkerID: 4 task_id: 37 執行中
worker 4 已完成task 37 ########
WorkerID: 10 task_id: 38 執行中
worker 10 已完成task 38 ########
WorkerID: 11 task_id: 39 執行中
worker 11 已完成task 39 ########
WorkerID: 12 task_id: 40 執行中
worker 12 已完成task 40 ########
WorkerID: 13 task_id: 41 執行中
worker 13 已完成task 41 ########
WorkerID: 7 task_id: 42 執行中
worker 7 已完成task 42 ########
WorkerID: 9 task_id: 43 執行中
worker 9 已完成task 43 ########
WorkerID: 6 task_id: 44 執行中
worker 6 已完成task 44 ########
WorkerID: 8 task_id: 45 執行中
worker 8 已完成task 45 ########
WorkerID: 5 task_id: 46 執行中
worker 5 已完成task 46 ########
WorkerID: 14 task_id: 47 執行中
worker 14 已完成task 47 ########
WorkerID: 0 task_id: 48 執行中
worker 0 已完成task 48 ########
WorkerID: 15 task_id: 49 執行中
worker 15 已完成task 49 ########
總計完成數量: 50
總計耗時: 21.207082327s
```
