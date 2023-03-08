# Solving Problem of Multithreading

## 1. Management of Shared Resources

## 2. DO NOT Lock Unkown Behavior
다음과 같은 클래스가 있다고 합시다.
```csharp
public class A {
    private static int a = 0;
    private object lock_object = new object();
    public void Add() {
        lock (lock_object) {
            a++;
        }
    }
}
```
이제 이를 사용할 때 어떤 사람이 이 클래스를 외부에서 사용할 때 이를 lock 한다고 가정합시다.
```csharp
public void SomeLogic() {
    A a = new();
    object lock_object = new object();
    lock (lock_object) {
        a.Add();
        // Do something
    }
}
```
이 과정에서 프로그래머는 자신이 잠금하는 것이 무엇인지 확인하지 않고 잠금하고 있습니다. 이미 `A.Add()` 메소드가 무엇을 하는지 모른다면 잠금을 하지 않는 것이 낫습니다. 이 코드에서는 `lock_object`가 서로 다르기 때문에 실제로 *교착상태*가 발생하지 않지만, 확인하지 않고 잠금을 걸어버리는 행위는 *교착상태*를 유발할 가능성이 있습니다.

## 3. Use Async-await Correctly
`HttpClient`로 웹 페이지를 받아오는 예제가 있다고 합시다. 
```csharp
internal class Program
{
    public async static Task<string> GetWebPageAsync(string url)
    {
        string content = await new HttpClient().GetStringAsync(url);
        Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
        return content;
    }

    static async Task Main(string[] args)
    {
        Stopwatch sw = new Stopwatch();
        sw.Start();
        string examplePage = await GetWebPageAsync("http://example.com");
        string googlePage = await GetWebPageAsync("http://google.com");
        sw.Stop();
        
        Console.WriteLine($"http://example.com: \n{examplePage}");
        Console.WriteLine($"http://google.com: \n{googlePage}");
        Console.WriteLine($"{sw.ElapsedMilliseconds}ms");
    }
}
```
```
from http://example.com, Thread ID: 6
from http://google.com, Thread ID: 7
http://example.com:
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

... 중략
686ms
```
아무런 문제가 없어 보입니다. 결과값도 잘 나오고 빠릅니다. 그런데 이 코드는 사실 `www.example.com`에서 페이지를 전부 받을때까지 대기하다가 `google.com`에서 페이지를 받아옵니다. 먼저 일을 시키고 작업이 끝나기를 기다려야 합니다. 다음은 좀 다르게 해봅시다.

```csharp
internal class Program
{
    public async static Task<string> GetWebPageAsync(string url)
    {
        string content = await new HttpClient().GetStringAsync(url);
        Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
        return content;
    }

    static async Task Main(string[] args)
    {
        Stopwatch sw = new Stopwatch();
        sw.Start();
        Task<string> exampleTask = GetWebPageAsync("http://example.com");
        Task<string> googleTask = GetWebPageAsync("http://google.com");

        string examplePage = await exampleTask;
        string googlePage = await googleTask;

        sw.Stop();
        
        Console.WriteLine($"http://example.com: \n{examplePage}");
        Console.WriteLine($"http://google.com: \n{googlePage}");
        Console.WriteLine($"{sw.ElapsedMilliseconds}ms");
    }
}
```
```
from http://example.com, Thread ID: 10
from http://google.com, Thread ID: 10
http://example.com:
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

... 중략
427ms
```
이제 대기없이 google 페이지를 받아옵니다. 단 실행결과에서 보았듯, 내부적으로 스레드 풀을 이용하기 떄문에 항상 다른 스레드가 작업을 실행한다는 것은 보장되지 않습니다. 실제로 다른 스레드가 실행하면 30ms에서 40ms가량 더 빠른 결과가 나옵니다. 위와 같은 내용은 async-await 패턴을 사용하는 모든 언어에서 적용됩니다.

물론 async-await 패턴을 사용한다고 모든 문제가 해결되는 것은 아닙니다. 공유 자원에 접근 할 때 여전히 *Race Condition*이 발생합니다.

## ABA Problem
잠금이나 CAS(Compare and Swap)과정에서 흔히 발생하는 문제입니다. 우선 .NET의 경우 GC를 사용하기 때문에 발생하는 문제는 아니지만[^1], 메모리를 직접 관리 하는 언어의 경우에는 이러한 문제가 발생합니다. 가령, `Atomic<T>`를 C++의 `std::atomic`과 거의 동치라고 생각합시다.

```csharp
class Node<T> {
    public Node<T> Next { get; set; }
    public T Value {get; set; }
}

class Stack<T> {
    private Atomic<Node<T>> top;

    public void Push(T element) {
        do {
            Node<T> current_top = top;
            element.Next = current_top;
        } while (!top.CompareAndExchange(current_top, element));
    }

    public T Pop() {
        Node<T> current_top;
        do {
            current_top = top;
            if (!current_top) {
                throw new Exception("Stack is empty");
            }

            Node<T> Next = current_top.Next;
        } while (!top.CompareAndExchange(current_top, next_top));
        return current_top;
    }
} 
```
이제 이 코드에서 두 개의 스레드가 돌고있다고 가정합시다.
1. $P_1$이 공유 메모리 자원에서 A값을 읽습니다.
2. $P_1$이 선점되고, $P_2$가 작동합니다.
3. $P_2$가 B값을 공유 메모리 자원에 씁니다.
4. $P_2$가 A값을 공유 메모리 자원에 씁니다.
5. $P_2$가 선점되고, $P_1$이 작동합니다.
6. $P_1$이 공유 메모리 자원에서 A값을 읽습니다.
7. $P_1$이 공유 메모리 자원의 값이 바뀐 적이 없다고 판단합니다.

분명히 중간에 상태가 바뀌었으나, 상태가 바뀌지 않았다고 판단합니다. 이것은 문제가 있습니다. 예를 들어 어디선가 동시에 Pop 연산을 수행했다고 합시다.
1. $P_1$이 Pop을 호출합니다.
```csharp
current_top = top;
if (!current_top) {
    throw new Exception("Stack is empty");
}

Node<T> Next = current_top.Next;
```
2. $P_2$가 Pop을 호출합니다.
```csharp
    current_top = top;
    if (!current_top) {
        throw new Exception("Stack is empty");
    }

    Node<T> Next = current_top.Next;
while (!top.CompareAndExchnage(current_top, next_top));
return top;
```
3. $P_2$가 정상적으로 값을 반환받았습니다.
4. $P_2$가 Push를 호출합니다.
```csharp
    Node<T> current_top = top;
    element.Next = current_top;
while (!top.CompareAndExchange(current_top, element));
```
5. $P_2$가 CAS를 정상적으로 통과하고 Push 함수를 마칩니다.
6. $P_1$이 CAS 연산을 수행합니다. CAS 연산이 성공합니다. 루프 밖으로 나갑니다.[^2]
7. 어?

원래라면 CAS 연산이 실패하고 다음 루프를 돌아야합니다. 하지만 이제 $P_1$과 $P_2$는 같은 객체를 사용합니다. 이는 *Race Condition*을 유발합니다.
## False sharing
캐시 구조와 관련이 있는 문제입니다. 여러 스레드가 동일한 캐시라인에 있는 변수를 업데이트하면 데이터 일관성을 위해 해당 캐시를 무효화시키고 메모리에 해당 값을 쓰고, 다시 읽는 작업을 수행합니다.

각 코어마다 고유한 캐시가 있는데 같은 캐시라인을 로드하는 경우가 있습니다. 이 때 두 데이터가 동일하지 않으면 (실제로는 어느쪽도 옳은 값이 아니지만) 무엇을 옳은 값으로 적용해야 하는지 알 수 없습니다. 2억 번의 루프를 도는 프로그램을 생각 해봅시다.
```csharp
static void Main(string[] args)
{
    int x = 0;
    const int iteration = 1_000_000_000;

    Stopwatch sw = new();
    sw.Start();

    Thread t1 = new(() => {
        for (int i = 0; i < iteration * 2; i++)
        {
            x++;
        }
        Console.WriteLine($"t1 {sw.ElapsedMilliseconds}ms");
    });
    t1.Start();
    t1.Join();
    sw.Stop();
}
```
```
t1 2965ms
```
이 작업을 둘로 나눠서 병렬화하면 2배의 속도 향상이 있을 것으로 예상됩니다.
```csharp
static void Main(string[] args)
{
    int x = 0, y = 0;
    const int iteration = 1_000_000_000;

    Stopwatch sw = new();
    sw.Start();

    Thread t1 = new(() => {
        for (int i = 0; i < iteration; i++)
        {
            x++;
        }
        Console.WriteLine($"t1 {sw.ElapsedMilliseconds}ms");
    });

    Thread t2 = new(() => {
        for (int i = 0; i < iteration; i++)
        {
            y++;
        }
        Console.WriteLine($"t2 {sw.ElapsedMilliseconds}ms");
    });

    t1.Start();
    t2.Start();
    t1.Join();
    t2.Join();
    sw.Stop();
}
```
```
t2 19111ms
t1 20296ms
```
무려 10배 이상 느려졌습니다. 이제 두 변수 `x`, `y`가 다른 캐시라인이 될 수 있도록 공간을 낭비해봅시다.
```csharp
static void Main(string[] args)
{
    const int cacheLineSize = 64;
    int[] variable = new int[2 * (cacheLineSize / sizeof(int))];
    const int iteration = 100_000_0000;
    const int x_location = 0;
    const int y_location = cacheLineSize / sizeof(int);

    Stopwatch sw = new();
    sw.Start();

    Thread t1 = new(() => {
        for (int i = 0; i < iteration; i++)
        {
            variable[x_location]++;
        }
        Console.WriteLine($"t1 {sw.ElapsedMilliseconds}ms");
    });

    Thread t2 = new(() => {
        for (int i = 0; i < iteration; i++)
        {
            variable[y_location]++;
        }
        Console.WriteLine($"t2 {sw.ElapsedMilliseconds}ms");
    });

    t1.Start();
    t2.Start();
    t1.Join();
    t2.Join();
    sw.Stop();
}
```
```
t2 1624ms
t1 1641ms
```
한 가지 재미있는 사실은 스레드를 여러 개로 늘리고 `cacheSize`를 조절하면, 시간이 거의 선형적으로 나타납니다. 예를 들어 이 작업을 4개로 분할하면 기존과 4배의 성능 향상이 일어납니다. 이 때 `cacheSize`를 절반으로 줄이면 대략적으로 10배의 성능 하락이 있습니다. 그래서 작업이 대략 `7500ms`정도가 걸림을 짐작할 수 있습니다. 이는 CPU 코어 수와 캐시라인 사이즈에 영향을 받아서, 천장효과가 있습니다.

그리고, 다른 변수가 아니라 같은 변수를 접근하더라도 캐시라인 일관성 유지를 위해서 쓰기연산을 하면 느려집니다. 0번 코어에서 0x1000에 쓰기 연산을 하고, 1번 코어에 0x1000이 캐시에 이미 로드되어 있다면 캐시가 무효화 됩니다. 이는 임계 영역 진입 후 공유 메모리에 값을 쓰는 행위가 생각보다 더 많은 비용을 치르는 이유 중 하나입니다.

## Data Parallelism
몇 가지 전제조건이 있습니다. 일단 연산의 결과과 이전 연산과 독립적이어야 합니다. 보통 행렬 덧셈 연산에서는 다음과 같이 실행합니다.
```csharp
public class Mat4 {
    private int array = new int[4][4];

    public void Add(Mat4 mat4) {
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                this.array[i][j] += mat4.array[i][j];
            }
        }
    }
}
```
그러나 이 문제를 잘 분석해보면 각 연산이 그 이전 연산과 독립적이라는 사실을 알 수 있습니다. 그렇다면 행이나 열로 문제를 나눌 수 있습니다. 행으로 나눠보면 다음과 같습니다.

```csharp
public class Mat4 {
    private int array = new int[4][4];

    public void Add(Mat4 mat4) {
        for (int i = 0; i < 4; i++) {
            ThreadPool.Run((i) => AddRow(i, mat4));
        }
        ThreadPool.WaitFor();
    }

    private void AddRow(int row, Mat4 mat4) {
        for (int i = 0; i < 4; i++) {
            this.array[row][i] = mat4[row][i];
        }
    }
}
```
이제 작업을 4개로 분할하였습니다. 이를 `ThreadPool`을 이용하여 실행하고 있습니다. 연산은 모두 동일하지만, 데이터를 다르게 사용합니다.

## Task Level Parallelism
상기 `Data Parallelism`과는 반대로 이쪽은 연산이 다른 경우입니다.
```csharp
public class Graphics {
    public void RunAndWait() {
        Task updateTask = Task(() => Update());
        Task drawTask = Draw(() => Draw());
        Task.WaitAll(updateTask, drawTask);
    }

    public void Update() {
        // Do update
    }

    public void Draw() {
        // Do draw
    }
}
```

## References
[1] https://en.wikipedia.org/wiki/ABA_problem

[^1] .NET, Java등의 언어에서는 자체 해결법이 존재합니다.

[^2] top이 가르키는 객체의 주소를 미리 받아왔었고, old value가 같은 값이니 통과 해버립니다.