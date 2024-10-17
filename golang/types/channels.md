### 1. [Функция merge](#merge-function)  
### 2. [Функция throttle](#throttle-function)  
### 3. [Функция calculateSum](#calculate-sum-function)  
### 4. [Функция fetch](#fetch-function)  
### 5. [Функция channels notification](#channels-notification-function)  
### 6. [Функция unpredictableFunc и predictable](#unpredictable-predictable-functions)  
### 7. [Функция semaphore](#semaphore-function)  
### 8. [Закрытие каналов в Go](#why-close-channels-in-go)

---

### Функция merge <a id="merge-function"></a>

```go
func merge(chans ...<-chan int) chan int {
	res := make(chan int)
	var wg sync.WaitGroup

	receiver := func(ch <-chan int) {
		defer wg.Done()

		for elem := range ch {
			res <- elem
		}
	}

	for _, ch := range chans {
		wg.Add(1)
		go receiver(ch)
	}

	go func() {
		wg.Wait()
		close(res)
	}()

	return res
}
```

**Описание**: Функция `merge` принимает несколько каналов на чтение (`<-chan int`) и объединяет их в один результирующий канал `res`, возвращая все значения из переданных каналов.

### Функция throttle <a id="throttle-function"></a>

```go
func throttle(f func(), ms time.Duration) func() {
	var lastTime time.Time

	return func() {
		now := time.Now()
		if now.Sub(lastTime) < ms {
			lastTime = now
			f()
		}
	}
}
```

**Описание**: Функция `throttle` принимает функцию `f` и ограничивает её вызовы через указанный промежуток времени `ms`. Полезно для ограничения частоты вызова функции.

### Функция calculateSum <a id="calculate-sum-function"></a>

```go
const n = 6

func calculateSum() {
	var res int
	chDone := make(chan int)
	chWait := make(chan struct{})
	defer func() {
		close(chWait)
		close(chDone)
	}()

	for i := 1; i <= n; i++ {
		go func(i int) {
			chDone <- i
		}(i)
	}

	go func() {
		var counter int
		for elem := range chDone {
			res += elem
			counter++
			if counter == n {
				chWait <- struct{}{}
			}
		}
	}()

	<-chWait

	fmt.Println(res)
}
```

**Описание**: Функция `calculateSum` запускает несколько горутин, каждая из которых отправляет значения в канал `chDone`. Как только все значения будут отправлены, подсчитывается сумма, и программа завершается.

### Функция fetch <a id="fetch-function"></a>

```go
func fetch(urls []string, limit int) {
	n := len(urls)
	if n < limit {
		limit = n
	}

	wg := sync.WaitGroup{}
	chURLs := make(chan string)

	wg.Add(limit)
	for i := 0; i < limit; i++ {
		go func() {
			defer wg.Done()
			for url := range chURLs {
				get(url)
			}
		}()
	}

	for _, url := range urls {
		chURLs <- url
	}
	close(chURLs)

	wg.Wait()
}

func get(url string) {
	resp, err := http.Get(url)
	if err != nil {
		log.Fatal("Error happened on request making")
	}
	defer resp.Body.Close()

	fmt.Println(resp.Status)
}
```

**Описание**: Функция `fetch` выполняет параллельные HTTP-запросы на список URL-адресов с ограничением на количество одновременно обрабатываемых запросов. Она использует каналы для передачи URL-адресов рабочим горутинам и синхронизирует их с помощью `sync.WaitGroup`.

### Функция channels notification <a id="channels-notification-function"></a>

```go
func main() {
	x := make(map[int]int, 1)
	var wg sync.WaitGroup

	ch1 := make(chan struct{})
	ch2 := make(chan struct{})
	ch3 := make(chan struct{})

	wg.Add(3)

	go func() {
		defer wg.Done()
		<-ch1
		x[1] = 2
		ch2 <- struct{}{}
	}()

	go func() {
		defer wg.Done()
		<-ch2
		x[1] = 5
		ch3 <- struct{}{}
	}()

	go func() {
		defer wg.Done()
		<-ch3
		x[1] = 10
	}()

	ch1 <- struct{}{}

	wg.Wait()

	fmt.Println("x[1] =", x[1]) // Expected to see 10
}
```

**Описание**: Функция `main` демонстрирует использование каналов и `WaitGroup` для синхронизации горутин, изменяющих значение в мапе `x`. Горутин выполнены последовательно, изменяя значение `x[1]` по мере передачи сигналов через каналы.

### Функция unpredictableFunc и predictable <a id="unpredictable-predictable-functions"></a>

```go
func main() {
	rand.NewSource(time.Now().UnixNano())
}

func unpredictableFunc() int64 {
	dur := rand.Int63n(5000)
	time.Sleep(time.Duration(dur) * time.Millisecond)
	return dur
}

func predictable(timeout time.Duration) int64 {
	ch := make(chan int64)
	go func() {
		ch <- unpredictableFunc()
	}()

	ctx, _ := context.WithTimeout(context.Background(), timeout)
	select {
	case dur := <-ch:
		return dur
	case <-ctx.Done():
		return 0
	}
}
```

**Описание**: Функция `unpredictableFunc` возвращает случайную длительность, задерживая выполнение программы на это время. Функция `predictable` использует контекст с тайм-аутом для предсказуемого выполнения, возвращая либо результат `unpredictableFunc`, либо 0, если произошел тайм-аут.

### Функция semaphore <a id="semaphore-function"></a>

```go
func worker(id int, sem chan struct{}) {
	sem <- struct{}{}
	
	fmt.Printf("Worker %d is done.\n", id)

	<-sem
}

func main() {
	const maxWorkers = 3
	sem := make(chan struct{}, maxWorkers)

	for i := 1; i <= 5; i++ {
		go worker(i, sem)
	}

	time.Sleep(10 * time.Second)
}
```

**Описание**: Функция `worker` использует семафор для ограничения количества одновременно выполняемых горутин. В `main` функции запускается 5 горутин, но только 3 из них могут выполняться одновременно, благодаря использованию буферизированного канала `sem`.

### Закрытие каналов в Go <a id="why-close-channels-in-go"></a>

**Причина закрытия каналов**:
1. **Избежание утечек горутин**: Закрытие каналов позволяет завершить горутины, которые ожидают данные из канала. Если канал не закрыт, горутины могут оставаться в блокированном состоянии.
2. **Оповещение через `range`**: При использовании `range` для чтения из канала цикл завершится, только если канал закрыт.
3. **Использование `select`**: Закрытие каналов помогает корректно завершать ожидание данных через конструкцию `select`.