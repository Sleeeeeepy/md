# Decorator Builder Pattern
우리는 객체의 책임을 동적으로 추가하기 위해서 다음과 같이 장식자 패턴을 이용합니다. 만약 그렇지 않다면 $m$개의 새로운 책임을 추가하기 위해서, $m!$개의 새로운 클래스를 생성해야 합니다.

```csharp
public interface IComponent {
    void draw();
}

public abstract class Decorator : IComponent {
    protected Component component;
    
    public Decorator(Component component) {
        this.component = component;
    }

    public virtual void draw() {
        this.component?.draw();
    }
}

public class BorderDecorator : Decorator {
	public BorderDecorator(IComponent component) : base(component) {}
    
    public override void draw() {
        base.draw();
        Console.WriteLine("Draw boarder");
    }
}

public class ScrollBarDecorator : Decorator {
    private int position = 0; 
	
	public ScrollBarDecorator(IComponent component) : base(component) {}

    public override void draw() {
        base.draw();
        Console.WriteLine("Draw scrollBar");
    }

    public void Scroll(int delta) {
        this.position += delta;
    }

    public void ListenEvent() {
        // ...
    }
}

public class TextView : IComponent {
    public void draw() {
        Console.WriteLine("Draw");
    }
}
```
잘 만들었으니 이제 클라이언트 코드도 만들어 봅시다.

```csharp
public class Program {
    public static void Main(string[] args) {
        IComponent textView = new BorderDecorator(new ScrollBarDecorator(new TextView()));

        textView.draw();
    }
}
```
`textView` 객체를 생성할 때 계속해서 코드가 길어집니다. 장식자가 한 개, 두 개면 괜찮지만, 네 개, 다섯 개가 되면 가독성이 심하게 떨어집니다. 이제 장식자 패턴에 빌더 패턴을 사용해봅시다.

```csharp
public class TextViewBuilder {
    private IComponent component = new TextView(); // head

    public TextViewBuilder AddScrollBar() {
        this.component = new ScrollDecorator(this.component);
        return this;
    }

    public TextViewBuilder AddBorder() {
        this.component = new BorderDecorator(this.component);
        return this;
    }

    public IComponent Build() {
        TextView textView = this.component;
        this.component = new TextView();
        return textView();
    }
}

public class Program {
    public static void Main(string[] args) {
        TextViewBuilder builder = new TextViewBuilder();
        IComponent textView = builder.AddScrollBar().AddBorder().Build();
        textView.draw();
    }
}
```
코드의 가독성이 훨씬 좋아졌습니다. 그런데 이와 같은 방법은 조금 단점이 있습니다. 예를 들어 `ScrollBar` 객체를 조작한다고 가정합니다. 기존 코드에서는 다음과 같이 사용합니다.

```csharp
public class Program {
    public static void Main(string[] args) {
        IComponent textView = new TextView();
        IComponent scrollBar = new ScrollBarDecorator(textView)
        IComponent border = new BorderDecorator(scrollbar);

        textView.draw();

        // 10만큼 스크롤
        scrollBar.Scroll(10);
    }
}
```
그러나 빌더 패턴의 사용 예제에서는 불가능합니다. 물론 방법이 아예 존재하지 않는 것은 아닙니다. C#에서는 `out`, `ref` 키워드를 사용해서 해결할 수 있습니다. 물론 그런 키워드가 없는 Java에서는 객체를 중간에 반환받는 메소드를 만들면 됩니다.

```csharp
public class TextViewBuilder {
    private IComponent component = new TextView(); // head

    public TextViewBuilder AddScrollBar() {
        this.component = new ScrollDecorator(this.component);
        return this;
    }

    public TextViewBuilder AddBorder() {
        this.component = new BorderDecorator(this.component);
        return this;
    }

    public IComponent Build() {
        TextView textView = this.component;
        this.component = new TextView();
        return textView();
    }

    public IComponent GetLastComponent() {
        return component;
    }
}

public class Program {
    public static void Main(string[] args) {
        TextViewBuilder builder = new TextViewBuilder();
        ScrollBarDecorator scrollbar = builder.AddScrollBar().GetLastComponent() as ScrollBarDecorator;
        TextView textView = builder.AddBorder().Build();
        textView.draw();

        // 10만큼 스크롤
        scrollBar.Scroll(10);
    }
}
```
