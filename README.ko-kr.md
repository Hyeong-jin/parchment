# Parchment [![Build Status](https://travis-ci.org/quilljs/parchment.svg?branch=master)](http://travis-ci.org/quilljs/parchment) [![Coverage Status](https://coveralls.io/repos/github/quilljs/parchment/badge.svg?branch=master)](https://coveralls.io/github/quilljs/parchment?branch=master)

Parchment는 [Quill](https://quilljs.com)의 문서 모델입니다. DOM 트리에 대한 병렬 트리 구조이며 Quill과 같은 내용 편집기에 유용한 기능을 제공합니다. Parchment 트리는 DOM 노드에 상응하는 [Blots](#blots)로 구성됩니다. Blot은 구조, 서식 및 / 또는 내용을 제공 할 수 있습니다. 또한 [Attributors](#attributors)는 간단한 서식 정보를 제공 할 수 있습니다.

**참고:** `new` 키워드로 Blot을 인스턴스화해서는 안됩니다. 이렇게하면 Blot의 필수 수명주기 기능을 방해 할 수 있습니다. [Registry](#registry)의 `create()` 메소드를 대신 사용하십시오.

`npm install --save parchment`

Quill이 Parchment의 문서 모델을 사용하는 방법에 대한 안내는 [Cloning Medium with Parchment](https://quilljs.com/guides/cloning-medium-with-parchment/)를 참조하십시오.

## Blots

Blot은 Parchment 문서의 기본 구성 요소입니다. [Block](#block-blot), [Inline](#inline-blot) 및 [Embed](#embed-blot)와 같은 몇 가지 기본 구현이 제공됩니다. 일반적으로 처음부터 빌드하는 대신 이 중 하나를 확장하는 것이 좋습니다. 구현 후 Blot을 사용하기 전에 등록([registered](#registry))해야합니다.

최소한 Blot은 정적 `blotName`으로 이름을 지정하고 `tagName` 또는 `className`과 연결해야합니다. Blot이 태그와 클래스 모두 정의 된 경우 클래스가 우선하지만 태그는 폴백으로 사용될 수 있습니다. Blot은 인라인인지 또는 블럭인지를 결정하는 [scope](#registry)도 가져야합니다.

```typescript
class Blot {
  static blotName: string;
  static className: string;
  static tagName: string;
  static scope: Scope;

  domNode: Node;
  prev: Blot;
  next: Blot;
  parent: Blot;

  // 해당 DOM 노드를 만듭니다
  static create(value?: any): Node;

  constructor(domNode: Node, value?: any);

  // leaves의 경우, blot의 값의 길이 ()
  // parents의 경우, 자식 값의 합
  length(): Number;

  // 주어진 인덱스와 길이 (해당되는 경우)를 조작합니다.
  // 적절한 자식으로 호출을 자주 전달합니다.
  deleteAt(index: number, length: number);
  formatAt(index: number, length: number, format: string, value: any);
  insertAt(index: number, text: string);
  insertAt(index: number, embed: string, value: any);

  // 이 Blot과 상위의 오프셋을 돌려줍니다.
  offset(ancestor: Blot = this.parent): number;

  // 업데이트 주기가 완료되면 호출됩니다.
  // 문서의 길이나 길이를 변경할 수 없으며
  // 모든 DOM 작업은 DOM 트래의 복잡성을 줄여야 합니다.
  optimize(): void;

  // Blot이 바뀌면 변이 기록으로 변경됩니다.
  // Blot 값의 내부 기록을 업데이트 할 수 있으며, Blot 자체의 수정이 허용됩니다.
  // 사용자 변경이나 API 호출로 트리거 될 수 있습니다.
  update(mutations: MutationRecord[]);


  /** Leaf Blots only **/

  // 이 Blot의 유형의 경우, domNode가 나타내는 값을 돌려줍니다.
  // domNode가 이 Blot 유형을 나타낼 수 있는지 확인하지 않아도 되므로
  // 응용프로그램은 호출하기 전에 외부에서 확인해야 합니다.
  static value(domNode): any;

  // Given location represented by node and offset from DOM Selection Range,
  // return index to that location.
  // 노드로 표현되는 위치와 DOM Selection Range의 오프셋이 주어지면
  // 그 위치에 대한 색인을 반환합니다.
  index(node: Node, offset: number): number;

  // Blot 내의 위치에 인덱스가 주어지면 DOM 선택 범위에서
  // 소모 할 수있는 해당 위치를 나타내는 노드와 오프셋을 반환합니다.
  position(index: number, inclusive: boolean): [Node, number];

  // 이 Blot이 나타내는 반환 값
  // update()에 의해 감지 될 수있는 API 또는
  // 사용자 변경과의 상호 작용없이 변경하면 안됩니다.
  value(): any;


  /** Parent blots only **/

  // 직접적인 자식이 될 수 있는 Blot의 화이트리스트 배열.
  static allowedChildren: Blot[];

  // Blot이 비어 있는 경우 삽입할 기본 자식 Blot.
  static defaultChild: string;

  children: LinkedList<Blot>;

  // 빌드 중에 호출되면, 자체의 자식 LinkedList를 채워야 합니다.
  build();

  // 후손을 위한 유용한 검색 함수, 수정하면 안됩니다
  descendant(type: BlotClass, index: number, inclusive): Blot
  descendents(type: BlotClass, index: number, length: number): Blot[];


  /** Formattable blots only **/

  // 이 Blot 유형인 경우 domNode가 나타내는 형식 값을 돌려줍니다.
  // domNode가 이 Blot 유형 인지 확인하지 않아도됩니다.
  static formats(domNode: Node);

  // Blot에 형식을 적용합니다. 자식이나 다른 blot에 옮겨서는 안됩니다.
  format(format: name, value: any);

  // 애트리뷰트를 포함하는 Blot이 나타내는 형식을 돌려줍니다.
  formats(): Object;
}
```

### Example

부모, 인라인 스코프 및 포맷 가능한 링크를 나타내는 Blot에 대한 구현입니다.

```typescript
import Parchment from 'parchment';

class LinkBlot extends Parchment.Inline {
  static create(url) {
    let node = super.create();
    node.setAttribute('href', url);
    node.setAttribute('target', '_blank');
    node.setAttribute('title', node.textContent);
    return node;
  }

  static formats(domNode) {
    return domNode.getAttribute('href') || true;
  }

  format(name, value) {
    if (name === 'link' && value) {
      this.domNode.setAttribute('href');
    } else {
      super.format(name, value);
    }
  }

  formats() {
    let formats = super.formats();
    formats['link'] = LinkBlot.formats(this.domNode);
    return formats;
  }
}
LinkBlot.blotName = 'link';
LinkBlot.tagName = 'A';

Parchment.register(LinkBlot);
```

Quill은 또한 [소스 코드](https://github.com/quilljs/quill/tree/develop/formats)에 많은 훌륭한 예제 구현을 제공합니다.

### Block Blot

블록 스코프 형식의 부모 Blot의 기본 구현. 기본적으로 블록 Blot을 포맷하면 Blot의 해당 하위 섹션이 바뀝니다.

### Inline Blot

인라인 범위가 지정된 형식화 가능한 부모 Blot의 기본 구현입니다. 기본적으로 인라인 Blot의 서식을 지정하면 다른 Blot이 자신을 감싸서 호출을 해당 하위 항목으로 전달합니다.

### Embed Blot

포맷이 가능한 비 텍스트 잎 Blot의 기본 구현. 해당 DOM 노드는 종종 [Void Element] (https://www.w3.org/TR/html5/syntax.html#void-elements)가되지만, [Normal Element] (https : // www .w3.org / TR / html5 / syntax.html # normal-elements). 이러한 경우 Parchment는 요소의 자식을 조작하거나 일반적으로 인식하지 않으므로 커서 / 선택 항목을 올바르게 사용하려면 Blot의 `index()` 및 `position()` 함수를 올바르게 구현하는 것이 중요합니다.

### Scroll

The root parent blot of a Parchment document. It is not formattable.
Parchment 문서의 루트 부모 Blot. 형식을 지정할 수 없습니다.

## Attributors

Attributor는 형식을 표현하는 대신 가볍고 경량의 대체 방법입니다. 그들의 DOM 상대는 [Attribute](https://www.w3.org/TR/html5/syntax.html#attributes-0)입니다. 노드에 대한 DOM 속성의 관계처럼, Attributors는 Blots에 속하게됩니다. [Inline](# inline-blot)이나 [Block](# block-blot) blot에서 `formats()`을 호출하면 대응하는 DOM 노드의 형식(있는 경우)과 DOM 노드의 속성이 나타내는 형식(있는 경우)을 돌려줍니다.

애트리뷰트의 인터페이스는 다음과 같습니다.

```typescript
class Attributor {
  attrName: string;
  keyName: string;
  scope: Scope;
  whitelist: string[];

  constructor(attrName: string, keyName: string, options: Object = {});
  add(node: HTMLElement, value: string): boolean;
  canAdd(node: HTMLElement, value: string): boolean;
  remove(node: HTMLElement);
  value(node: HTMLElement);
}
```

참고로 사용자 정의 Attributor는 Blots와 같은 클래스 정의가 아닌 인스턴스입니다. Blot과 마찬가지로 처음부터 새로 만드는 대신 Base [Attributor](#attributor), [Class Attributor](#class-attributor) 또는 [Style Attributor](#style-attributor) 같은 기존 구현을 사용하는 것이 좋습니다.

Attributor에 대한 구현은 놀라울 정도로 간단하며 [소스 코드](https://github.com/quilljs/parchment/tree/master/src/attributor)는 다른 이해의 근원이 될 수 있습니다.

### Attributor

일반 속성을 사용하여 형식을 나타냅니다.

```js
import Parchment from 'parchment';

let Width = new Parchment.Attributor.Attribute('width', 'width');
Parchment.register(Width);

let imageNode = document.createElement('img');

Width.add(imageNode, '10px');
console.log(imageNode.outerHTML);   // Will print <img width="10px">
Width.value(imageNode);	                // Will return 10px
Width.remove(imageNode);
console.log(imageNode.outerHTML);   // Will print <img>
```

### Class Attributor

classname 패턴을 사용하여 형식을 나타냅니다.

```js
import Parchment from 'parchment';

let Align = new Parchment.Attributor.Class('align', 'blot-align');
Parchment.register(Align);

let node = document.createElement('div');
Align.add(node, 'right');
console.log(node.outerHTML);  // Will print <div class="blot-align-right"></div>
```

### Style Attributor

인라인 스타일을 사용하여 형식을 나타냅니다.

```js
import Parchment from 'parchment';

let Align = new Parchment.Attributor.Style('align', 'text-align', {
  whitelist: ['right', 'center', 'justify']   // Having no value implies left align
});
Parchment.register(Align);

let node = document.createElement('div');
Align.add(node, 'right');
console.log(node.outerHTML);  // Will print <div style="text-align: right;"></div>
```

## Registry

모든 방법은 Parchment ex에서 액세스 할 수 있습니다. 예. `Parchment.create('bold')`.

```typescript
// 이름 또는 DOM 노드가 지정된 blot을 작성합니다.
// 범위가 주어지면 범위와 동일한 이름을 만듭니다.
create(domNode: Node, value?: any): Blot;
create(blotName: string, value?: any): Blot;
create(scope: Scope): Blot;

// 이름 또는 DOM 노드가 지정된 Blot을 작성합니다.
// Bubbling은 해당 Embed Blot을 검색 할 때 유용합니다.
// DOM 노드의 자손 노드.
find(domNode: Node, bubble: boolean = false): Blot;

// Blot 또는 Attributor 검색
// 범위가 주어지면 범위와 이름이 같은 얼룩을 찾습니다.
query(tagName: string, scope: Scope = Scope.ANY): BlotClass;
query(blotName: string, scope: Scope = Scope.ANY): BlotClass;
query(domNode: Node, scope: Scope = Scope.ANY): BlotClass;
query(scope: Scope): BlotClass;
query(attributorName: string, scope: Scope = Scope.ANY): Attributor;

// Blot 클래스 정의 또는 Attributor 인스턴스를 등록합니다.
register(BlotClass | Attributor);
```
