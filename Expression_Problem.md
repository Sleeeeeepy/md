# Expression Problem

```fsharp
type ImageProcess =
    | JPEGProcess
    | PNGProcess

let resize process =
    match process with
    | JPEGProcess ->
        printfn "resize JPEG"
    | PNGProcess ->
        printfn "resize PNG"
```
이제 새로운 이미지 `GIF`를 추가하면 기존 코드를 수정 해야합니다. 반면 *C#* 코드를 봅시다.

```csharp
public interface ImageProcess {
    void resize();
}

public class JPEGProcess : ImageProcess {
    void resize() {
        Console.WriteLine("resize JPEG");
    }
}

public class PNGProcess : ImageProcess {
    void resize() {
        Console.WriteLine("resize PNG");
    }
}
```
새로운 이미지 `GIF`를 기존 코드의 수정없이 추가할 수 있습니다. 하지만 새로운 연산 `grayscale`을 추가하려면 기존 코드를 수정 해야합니다.

이렇듯, 함수형 프로그래밍 언어에서는 새로운 행위를 추가하는 것은 쉽지만 새로운 타입을 추가하는 것은 어렵습니다. 반면, 객체 지향 언어에서는 새로운 타입을 추가하는 것은 쉽지만, 새로운 행위를 추가하는 것이 어렵습니다. 이는 어느 쪽에서든 해결하기 어려운 문제 중 하나입니다.

## 6. Soluntion : Multiple Dispatch

### 6-1. Dispatch
디스패치는 특정 함수 호출로 인해 실행될 코드를 결정하는 과정입니다. 여기에는 상속, 다형성 또는 동적 바인딩과 같은 다양한 기술을 사용하여 실행할 코드를 결정할 수 있습니다. 다음과 같은 코드에서 `MakeSound()`를 호출하면 어떤 코드가 실행될까요?
``` csharp
public class Animal {
    public virtual void MakeSound() {
        Console.WriteLine("알 수 없는 동물소리");
    }
}

public class Dog : Animal {
    public override void MakeSound() {
        Console.WriteLine("멍멍");
    }
}

public class Cat : Animal {
    public override void MakeSound() {
        Console.WriteLine("야옹");
    }
}

public class Program {
    static void Main(string[] args) {
        Animal dog = new Dog();
        dog.MakeSound();

        Animal cat = new Cat();
        cat.MakeSound();
    }
}
```
```
멍멍
야옹
```

만약 `virtual` 키워드를 생략한다면 베이스 클래스 `Animal`의 `MakeSound()`가 출력됩니다. 이와같이 호출할 함수를 결정하는 일련의 과정을 디스패치라고 합니다.

### 6-2. Double Dispatch
그러나 함수 오버로딩을 이용하면 어떤 문제가 생길까요? 비슷하지만 약간 다른 예제입니다.

```csharp
public class Animal {
    public virtual void MakeSound() {
        Console.WriteLine("알 수 없는 동물소리");
    }
}

public class Dog : Animal {
    public override void MakeSound() {
        Console.WriteLine("멍멍");
    }
}

public class Cat : Animal {
    public override void MakeSound() {
        Console.WriteLine("야옹");
    }
}

public class SoundListener {
    public void Listen(Animal animal) {
        Console.WriteLine("알 수 없는 동물 소리를 들었다.");
    }

    public void Listen(Cat cat) {
        Console.WriteLine("고양이 소리를 들었다.");
    }

    public void Listen(Dog dog) {
        Console.WriteLine("강아지 소리를 들었다.");
    }
}

public class Program {
    static void Main(string[] args) {
        SoundListener listener = new SoundListener();
        Animal dog = new Dog();
        Animal cat = new Cat();

        listener.Listen(dog);
        listener.Listen(cat);
    }
}
```
```
알 수 없는 동물 소리를 들었다.
알 수 없는 동물 소리를 들었다.
```
cat은 Cat 타입임에도 불구하고 `SoundListener.Listen(Animal others)`를 호출합니다. 이 문제를 해결하려면 더블 디스패치 기법을 이용합니다.

```csharp
public class Animal {
    public virtual void MakeSound() {
        Console.WriteLine("알 수 없는 동물소리");
    }

    public virtual void Accept(SoundListener listener) {
        listener.Listen(this);
    }
}

public class Dog : Animal {
    public override void MakeSound() {
        Console.WriteLine("멍멍");
    }

    public override void Accept(SoundListener listener) {
        listener.Listen(this);
    }
}

public class Cat : Animal {
    public override void MakeSound() {
        Console.WriteLine("야옹");
    }

    public override void Accept(SoundListener listener) {
        listener.Listen(this);
    }
}

public class SoundListener {
    public void Listen(Animal animal) {
        Console.WriteLine("알 수 없는 동물 소리를 들었다.");
    }

    public void Listen(Cat cat) {
        Console.WriteLine("고양이 소리를 들었다.");
    }

    public void Listen(Dog dog) {
        Console.WriteLine("강아지 소리를 들었다.");
    }
}

public class Program {
    static void Main(string[] args) {
        SoundListener listener = new SoundListener();
        Animal dog = new Dog();
        Animal cat = new Cat();

 		dog.Accept(listener);
		cat.Accept(listener);
    }
}
```
```
강아지 소리를 들었다.
고양이 소리를 들었다.
```
더블 디스패치를 통해 문제를 해결했습니다. 방문자 패턴이 바로 위와 같이 더블 디스패치를 사용하는 예제입니다. 사실, 방문자 패턴과 동일합니다. 따라서 이 코드에는 문제가 있습니다. 지원하는 동물의 종류를 늘리려면 `SoundListener`를 반드시 수정해야합니다.

### 6-3. Solution: Multiple Dispatch
타입에 대해 약간만 포기하면, 표현 문제를 해결할 수 있습니다.

``` csharp
public class Animal {
    public virtual void MakeSound() {
        Console.WriteLine("알 수 없는 동물소리");
    }
}

public class Dog : Animal {
    public override void MakeSound() {
        Console.WriteLine("멍멍");
    }
}

public class Cat : Animal {
    public override void MakeSound() {
        Console.WriteLine("야옹");
    }
}

public class SoundListener {
    public static void Listen(Animal animal) {
        Console.WriteLine("알 수 없는 동물 소리를 들었다.");
    }

    public static void Listen(Cat cat) {
        Console.WriteLine("고양이 소리를 들었다.");
    }

    public static void Listen(Dog dog) {
        Console.WriteLine("강아지 소리를 들었다.");
    }
}

public class Program {
    static void Main(string[] args) {
        Animal dog = new Dog();
        Animal cat = new Cat();

		void Listen(Animal animal) => SoundListener.Listen(animal as dynamic);
		Listen(dog);
		Listen(cat);
    }
}
```
이제 새로운 동물, 닭을 추가 해봅시다.
``` csharp
// ...
public class Chicken: Animal {
    public override void MakeSound() {
        Console.WriteLine("꼬끼오");
    }
}

public class SoundListener {
    // ...
    public static void Listen(Chicken chicken) {
        Console.WriteLine("닭 소리를 들었다.");
    }
}

public class Program {
    static void Main(string[] args) {
        Animal dog = new Dog();
        Animal cat = new Cat();
        Animal chicken = new Chicken();
        
		void Listen(Animal animal) => SoundListener.Listen(animal as dynamic);
		Listen(dog);
		Listen(cat);
        Listen(chicken);
    }
}
```
클래스 `SoundListener`를 수정했으므로 **OCP**를 위반한 것처럼 보입니다. 그런데, 사실 SoundListener의 역할은 사실상 존재하지 않습니다. 이제 지워봅시다.


``` csharp
public class Program {
    static void Listen(Animal animal) {
        Console.WriteLine("알 수 없는 동물 소리를 들었다.");
    }

    static void Listen(Cat cat) {
        Console.WriteLine("고양이 소리를 들었다.");
    }

    static void Listen(Dog dog) {
        Console.WriteLine("강아지 소리를 들었다.");
    }

    static void Listen(Chicken chicken) {
        Console.WriteLine("닭 소리를 들었다.");
    }

    static void Main(string[] args) {
        Animal dog = new Dog();
        Animal cat = new Cat();
        Animal chicken = new Chicken();

		Listen(dog as dynamic);
		Listen(cat as dynamic);
        Listen(chicken as dynamic);
    }
}
```
이제 타입을 추가하는데 아무런 이상이 없다는 것이 명확합니다. 물론, 새로운 연산을 추가하는 것도 아무런 문제가 없습니다. 동적 타입이 약간 불안하긴 하지만[^1] **OCP**를 준수합니다. 문제를 완벽히 해결했습니다.

한 가지 재미있는 사실은 이러한 방법이 초기 객체 지향 언어에서 주로 사용되던 방식이라는 사실입니다. 이런 식의 사용이 OO의 원조랑 가장 비슷합니다. 현대의 객체 지향 언어에서는 메세징보다는, 클래스와 객체 자체에 집중합니다. 사실 객체 지향 프로그래밍은 상속을 이용한 코드 재사용을 강조하기보다는, 캡슐화 그 자체를 강조했습니다.
# 8. Solution : Object Algebras
이제, 다음과 같은 인터페이스가 있다고 가정 해봅시다.
```csharp
public interface ExpressionAlgebra<E> {
    E IntegerLiteral(int n); // literal node is terminal node.
    E AddExpression(E left, E right); // add node is non-terminal node. 
}
```
그리고 이를 평가하는 evaluator를 하나 만들어 봅시다.
```csharp
public class DefaultEvaluator : ExpressionAlgebra<int> {
    public int IntegerLiteral(int n) {
        return n;
    }

    public int AddExpression(int left, int right) {
        return left + right;
    } 
}
```
시간이 조금 흘러서, 문법을 출력하고 싶습니다. 그래서 새로운 클래스를 하나 만듭시다.
```csharp
public class Print : ExpressionAlgebra<string> {
    public string IntegerLiteral(int n) {
        return "" + n;
    }

    public string AddExpression(string left, string right) {
        return left + " + " + right;
    }
}
```
우리는 기존 클래스를 전혀 수정하지 않고 새로운 기능을 추가했습니다. 이제 새로운 문법을 추가한다고 가정합시다. 이제 곱하기를 추가합시다. 그렇다면 곱하기 표현식을 추가해야합니다.
```csharp
public interface MultiplyAlgebra<E> : ExpressionAlgebra<E> {
    E MultiplyExpression(E left, E right);
}
```
이를 인터페이스 상속으로 해결합니다. 그렇다면, 우리가 이전에 만들었던 `DefualtEvaluator`와 `Print`는 어떻게 될까요? 당연히 아무런 문제가 없습니다. 이제 이를 평가하는 evaluator도 한 번 만들어 봅시다.
```csharp
public class MultiplyEvaluator : DefaultEvaluator, MultiplyAlgebra<int> {
    public MultiplyExpression(int left, int right) {
        return left * right;
    }
}
```
이제, `Print`에 대해서도 새로운 문법을 처리하도록 해야합니다.
```csharp
public class MultiplyPrint : Print, MultiplyAlgebra<string> {
    public MultiplyExpression(string left, string right) {
        return left + "*" + right;
    }
}
```
클라이언트 코드를 만들어 봅시다.
``` csharp
public E MakeExpression<E>(ExpressionAlgebra<E> algebra) {
    return algebra.AddExpression(algebra.IntegerLiteral(2),
                                 algebra.IntegerLiteral(3));
}
```
만약 곱하기를 지원하려면 `ExpressionAlgebra<E>` 부분을 `MutiplyAlgebra<E>`로 변경하면 됩니다. 우리는 기존 기능을 수정하지 않고 새로운 문법을 추가했습니다. 여기서 `MultiplyAlgebra<E>`을 추가한 것은 타입을 추가하는 것으로 생각할 수 있습니다. 또한 `Print`를 추가한 것은 연산을 추가하는 것으로 생각할 수 있습니다. 우리는 타입과 연산의 확장에 대해서 열려있습니다. 수정에는 닫혀있습니다. **OCP**를 만족합니다.

이제 연산을 수평적으로 확장해봅시다.

```csharp
public Pair<A, B> {
    public A a {get; private set;}
    public B b {get; private set;}

    public Pair(A a, B b) {
        this.a = a;
        this.b = b;
    }
}

public class MultipleTypeAlgebra<A, B> : ExpressionAlgebra<Pair<A, B>> {
    ExpressionAlgebra<A> e1;
    ExpressionAlgebra<B> e2;

    public Pair<A, B> IntegerLiteral(int n) {
        return new Pair<A, B>(e1.IntegerLiteral(n), e2.IntgerLiteral(n));
    }

    public Pair<A, B> AddExpression(Pair<A, B> left, Pair<A, B> right) {
        return new Pair<A, B>(e1.AddExpression(left.a, right.a), 
                              e2.AddExpression(left.b, right.b));
    }
}
```
연산이 수평적으로 확장됐습니다. 이제 Print와 DefualtEvaluator를 동시에 사용할 수 있습니다.

# 9. Happy Ending?
그런데, 둘 중 한 가지 방법을 택해서 구현을 하면 아무런 문제가 없을까요? 아닙니다. 각 방법에는 단점이 존재합니다. 동적 타입의 사용은 디버깅을 어렵게 합니다. 객체 대수는 복잡하고 배우기 어려우며 코드를 지저분하게 합니다.[^2]

사실 *Object Algebras*를 사용하지 않더라도 문제를 해결할 수 있는 방법이 있습니다.
```csharp
public interface Expression() {
    int Evaluation();
}

public class IntegerLiteral : Expression
{
    public int Value { get; }

    public IntegerLiteral(int value)
    {
        Value = value;
    }

    public int Evaluation() {
        return Value;
    }
}

public class AddExpression : Expression
{
    public Expression Left { get; }
    public Expression Right { get; }

    public AddExpression(Expression left, Expression right)
    {
        Left = left;
        Right = right;
    }

    public int Evaluation() {
        return Left.Evaluation() + Right.Evaluation();
    }
}
```
이제 새로운 Expression을 추가해봅시다.
```csharp
public class MultiplyExpression : Expression {
    public Expression Left { get; }
    public Expression Right { get; }

    public MultiplyExpression(Expression left, Expression right) {
        Left = left;
        Right = right;
    }

    public int Evaluation() {
        return Left.Evaluation() * Right.Evaluation();
    }
}
```
이제 `Print` 행위를 추가해봅시다.
```csharp
interface PrintableExpression : Expression {
    void print();
}

public class PrintableAddExpression : AddExpression, PrintableExpression {
    public void Print() {
        Console.WriteLine($"{Left.Print()} + {Right.Print()}");
    }
}
// ... 하략
```
그런데 여기에는 문제가 있습니다. `PrintableAddExpression`의 `Left`와 `Right` 속성이 `Expression` 타입이기 때문에 `PrintableExpression.print()`를 실행하다가 일반 `AddExpression`을 만나면 함수를 찾을 수 없어서 예외가 발생할 것입니다. 이는 제네릭 제약조건으로 보완할 수 있습니다.
```csharp
public AddExpression<E> : Expression where E : Expression {
    public E Left { get; }
    public E Right { set; }
    // ...
}

public PrintableAddExpression<E> : AddExpression<E>, PrintableExpression where T : Printable Expression {
    // ...
}
```
[^1]: 만약 Animal이 아닌 Tree가 온다고 생각해봅시다. 그렇다면 호출할 함수를 결정할 수 없습니다.

[^2]: 상용구 코드가 너무 많아집니다.

# Appendix A. Using Visitor Pattern Is Another Alternative
```csharp
public interface Expression<A> {
    A Accept(IExpressionVisitor<A> visitor); 
}

public abstract class IExpressionVisitor<A> {
    public virtual A VisitExpression(Expression expression) {
        if (expression == null) return default(A);
    }

    public virtual A VisitDefault(Expression expression) {
        // 기본 행위를 지정합시다. 다른 노드를 더 방문하게 할 수도 있습니다.
        return default(A);
    }

    public virtual A VisitAddExpression(AddExpression expression) {
        Debug.WriteLine($"call VisitAddExpression but {expression.getType()} is not implemented.");
        return this.VisitDefault(expression);
    }
}

public class AddIntegerExpression : Expression<int> {
    // ...

    public int Accept(IExpressionVisitor<int> visitor) {
        return visitor.VisitAddExpression(this);
    }
}
```
이렇게 만들면, 문제를 완전히 해결한 것은 아니지만 방문자 패턴을 조금 더 유연하게 사용할 수 있습니다.

# Appendix B. Rust
러스트에서는 이 문제를 어떻게 해결 했을까요? 러스트에서는 `trait`를 이용하여 해결합니다.
```rust
// rust에서는 클래스도 없고 상속도 없다.
struct Cat {

}

trait MakeSound {
    fn make_sound(&self);
}

// Cat에 make_sound 함수를 추가해보자.
impl MakeSound for Cat {
    fn make_sound(&self) {
        println("야옹");
    }
}

struct Dog {

}

impl MakeSound for Dog {
    fn make_sound(&self) {
        println!("멍멍");
    }
}

trait Listen {
    fn listen(&self);
}

// Dog에 listen 함수를 추가해보자.
impl Listen for Dog {
    fn listen(&self) {
        println!("강아지 소리를 들었다.");
    }
}

// Cat에도 구현을 하자.
impl Listen for Cat {
    fn listen(&self) {
        println!("고양이 소리를 들었다.");
    }
}

// 정적 디스패치
fn listen<T: Listen>(animal: T) {
    animal.listen();
}

// 다이나믹 디스패치
fn make_sound(animal: &dyn MakeSound) {
    animal.make_sound();
}
```

# References
[1] Extensibility for the masses: Practical extensibility with object algebras
BCS Oliveira, WR Cook ECOOP 2012–Object-Oriented Programming: 26th European Conference, Beijing, China, June 11-16, 2012. Proceedings 26, 2-27, 2012

[2] Wang, Yanlin & Oliveira, Bruno. (2016). The expression problem, trivially!. 37-41. 10.1145/2889443.2889448.