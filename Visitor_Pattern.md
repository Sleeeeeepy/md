# Visitor Pattern And Expression Problem
## 1. Situation
다음과 같이 특정 이미지를 처리하는 프로그램이 있다고 생각해봅시다.
```typescript
interface IImageProcess {
    resize(): void;
}

class JPEGImageProcess implements IImageProcess {
    public resize(): void {
        console.log("resize JPEG\n");
    }
}

class PNGImageProcess implements IImageProcess {
    public resize(): void {
        console.log("resize PNG\n");
    }
}
```
그런데, 시간이 지나 새로운 이미지 파일을 지원해야 합니다. 이제 WebP 파일을 지원 해봅시다.

```typescript
class WebPImageProcess implements IImageProcess {
    public resize(): void {
        console.log("resize WebP\n");
    }
}
```
아주 좋습니다. 주어진 문제를 완벽히 해결 했습니다. 그런데 시간이 조금 더 흘러서, 이제는 이미지를 그레이스케일하고자 합니다. 그러면 이제 `interface IImageProcess`를 수정해야 합니다.

```typescript
interface IImageProcess {
    resize(): void;
    grayscale(): void;
}
```
이는 명백히 **OCP**를 위반합니다. 이제 IImageProcess를 구현하는 모든 클래스를 수정 해야합니다. 인터페이스를 두개로 쪼개볼까요? `resize()` 혼자 있기에는 인터페이스 이름이 어색하니 이름도 바꿔봅시다.

```typescript
interface IResizeProcess {
    resize(): void;
}

interface IGrayscaleProcess {
    garyscale(): void;
}
```
하지만 이렇게 만든다고 문제가 해결되는 것도 아닐 뿐더러, 또 다른 문제가 발생합니다. $m$개의 이미지 타입과 $n$개의 연산에 대해 새로운 이미지 타입을 지원하거나, 새로운 연산을 지원해야 할 경우 $n$개의 인터페이스와 $mn$개의 클래스를 생성하고 관리 해야합니다. 이 클래스들을 생성하는 부분이 문제에 시달립니다. 리스크를 생성하는 부분에 전가했습니다.

타입과 행위에 대한 변경이 잦다면 코드를 관리하기 어렵습니다. 하지만 대부분의 애플리케이션에서는 새로운 기능 하나를 추가하기 위해서 새로운 타입을 만들거나, 새로운 연산을 만들거나 둘 중 하나의 변경사항만 처리하면 됩니다.

이 응용 프로그램의 경우, 새로운 이미지 포맷의 도입은 드뭅니다. 대부분의 이미지 포맷을 지원하고 나면 적어도 몇 년 동안은 문제가 없을 것입니다. 새로운 포맷을 추가하는 것은 몇 년에 한번이면 충분합니다. 하지만 이미지 연산은 며칠을 간격으로 추가된다고 생각합시다. 인터페이스를 수정하고 모든 클래스를 수정합니다. 작업을 전부 끝내기 전까지는 컴파일도 테스트도 불가능합니다. 문제가 있습니다.

## 2. Visitor Pattern 적용하기
방문자 패턴을 적용하면 **OCP**를 위반하지 않고 새로운 기능을 추가할 수 있습니다.

```typescript
interface IImageVisitor {
    visitJPEG(img: JPEG): void;
    visitPNG(img: PNG): void;
    visitWebP(img: WebP): void;
}

interface Image {
    accept(visitor: IImageVisitor);
}

class ImageResizeVisitor implements IImageVisitor {
    visitJPEG(img: JPEG): void {
        console.log("resize JPEG\n");
    }

    visitPNG(img: PNG): void {
        console.log("resize PNG\n");
    }

    visitWebP(img: WebP): void {
        console.log("resize WebP\n");
    }
}

class ImageGraysacleVisitor implements IImageVisitor {
    visitJPEG(img: JPEG): void {
        console.log("grayscale JPEG\n");
    }

    visitPNG(img: PNG): void {
        console.log("grayscale PNG\n");
    }

    visitWebP(img: WebP): void {
        console.log("grayscale WebP\n");
    }
}

class ImageCropVisitor implements IImageVisitor {
    visitJPEG(img: JPEG): void {
        console.log("crop JPEG\n");
    }

    visitPNG(img: PNG): void {
        console.log("crop PNG\n");
    }

    visitWebP(img: WebP): void {
        console.log("crop WebP\n");
    }
}

class JPEG implements Image {
    accept(visitor: IImageVisitor): void {
        visitor.visitJPEG(this);
    }
}

class PNG implements Image {
    accept(visitor: IImageVisitor): void {
        visitor.visitPNG(this);
    }
}

class WebP implements Image {
    accept(visitor: IImageVisitor): void {
        visitor.visitWebP(this);
    }
}
```
새로운 기능 `crop`을 추가하기 위해서 `ImageCropVisitor`를 새로 만들었습니다. 기능을 추가하는 것은 기존 클래스를 전혀 수정하지 않습니다. 이제 **OCP**를 만족합니다. 



## 3. Variation

방문자 패턴은 주로 컴포지트 패턴과 결합하여 사용합니다. 가장 유명한 예제로는 컴파일러가 있습니다. 컴파일러는 타입 체크와 코드 생성 시 컴포지트 패턴으로 구성된 구문 트리를 방문자 패턴을 이용합니다. 방문자 패턴을 이용하는 컴파일러에는 *Roslyn*[^1]과 *rustc*가 있습니다. 이외에도 수많은 컴파일러에서 방문자 패턴을 사용합니다.

## 4. Result

1. 새로운 이미지를 추가하는 것은 어려워졌지만, 새로운 연산을 추가하는 것은 쉬워졌습니다.
2. 데이터 은닉이 어렵습니다. 원하는 작업을 수행하기 위해 필요한 연산들을 외부로 노출시켜야 합니다.
3. 외부 라이브러리 등, 기존의 코드를 수정하기 어렵다면 패턴을 적용하기 어렵습니다. 문제를 해결하기 위한 충분한 정보와 연산을 이로부터 제공받을 수 있다면, Wrapper class를 작성하는 미봉책이 있습니다.

## 5. See also

[Expression Problem]\: 방문자 패턴을 사용하면 타입 추가의 용이성과 연산 추가의 용이성이 교환됩니다. 일반적으로 객체 지향 프로그래밍에서는 타입 추가가 용이하고 연산 추가가 어렵습니다. 반면, 함수형 프로그래밍에서는 연산 추가가 용이하고 타입 추가가 어렵습니다. 양 쪽 패러다임에서 서로 반대의 상황을 해결하기 어렵습니다. 이를 **표현 문제**라고 합니다.

[Expression Problem]: ./Expression_Problem.md

## FOOT NOTES

[^1]: 물론 Roslyn은 Syntax.xml을 통해 자동으로 생성된 virtual로 선언된 방문자를 사용합니다. 문법의 변경이 잦은 C#의 특성을 고려한 것입니다. 약간 맛만 봅시다. Roslyn의 한가지 놀라운 점은 자동으로 생성된 코드가 사람이 읽기 쉽게 되어있다는 것 입니다.
```xml
  <Node Name="IdentifierNameSyntax" Base="SimpleNameSyntax">
    <Kind Name="IdentifierName"/>
    <Field Name="Identifier" Type="SyntaxToken" Override="true">
      <Kind Name="IdentifierToken"/>
      <Kind Name="GlobalKeyword"/>
      <PropertyComment>
        <summary>SyntaxToken representing the keyword for the kind of the identifier name.</summary>
      </PropertyComment>
    </Field>
    <TypeComment>
      <summary>Class which represents the syntax node for identifier name.</summary>
    </TypeComment>
    <FactoryComment>
      <summary>Creates an IdentifierNameSyntax node.</summary>
    </FactoryComment>
  </Node>
```
정의가 여기에 있고, 이것은 다음 코드를 생성합니다.

```csharp
    public sealed partial class IdentifierNameSyntax : SimpleNameSyntax
    {

        internal IdentifierNameSyntax(InternalSyntax.CSharpSyntaxNode green, SyntaxNode? parent, int position)
          : base(green, parent, position)
        {
        }

        /// <summary>SyntaxToken representing the keyword for the kind of the identifier name.</summary>
        public override SyntaxToken Identifier => new SyntaxToken(this, ((Syntax.InternalSyntax.IdentifierNameSyntax)this.Green).identifier, Position, 0);

        internal override SyntaxNode? GetNodeSlot(int index) => null;

        internal override SyntaxNode? GetCachedSlot(int index) => null;

        public override void Accept(CSharpSyntaxVisitor visitor) => visitor.VisitIdentifierName(this);
        public override TResult? Accept<TResult>(CSharpSyntaxVisitor<TResult> visitor) where TResult : default => visitor.VisitIdentifierName(this);

        public IdentifierNameSyntax Update(SyntaxToken identifier)
        {
            if (identifier != this.Identifier)
            {
                var newNode = SyntaxFactory.IdentifierName(identifier);
                var annotations = GetAnnotations();
                return annotations?.Length > 0 ? newNode.WithAnnotations(annotations) : newNode;
            }

            return this;
        }

        internal override SimpleNameSyntax WithIdentifierCore(SyntaxToken identifier) => WithIdentifier(identifier);
        public new IdentifierNameSyntax WithIdentifier(SyntaxToken identifier) => Update(identifier);
    }
```
혹시 *Rolsyn*에서 문법을 추가하는 과정이 궁금하시다면 [여기](https://smellegantcode.wordpress.com/2014/04/27/adventures-in-roslyn-a-new-kind-of-managed-lvalue-pointer/)와 [여기](https://smellegantcode.wordpress.com/2014/04/26/adventures-in-roslyn-using-pointer-syntax-as-a-shorthand-for-ienumerable/)를 클릭해주세요. C#에 C 포인터 문법을 추가하는 방법에 대한 블로그 글입니다. [아니면 새로운 연산자를 추가하는 글도 있습니다.](https://marcinjuraszek.com/2017/05/adding-matt-operator-to-roslyn-part-1.html) 연산자를 추가하기 이전에 [새로운 인스트럭션을 coreclr에 추가하는 방법도 있습니다.](https://mattwarren.org/2017/05/19/Adding-a-new-Bytecode-Instruction-to-the-CLR/#step-0)