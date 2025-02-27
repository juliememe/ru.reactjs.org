---
id: portals
title: Порталы
permalink: docs/portals.html
---

Порталы позволяют рендерить дочерние элементы в DOM-узел, который находится вне DOM-иерархии родительского компонента.

```js
ReactDOM.createPortal(child, container)
```

Первый аргумент (`child`) — это [любой React-компонент, который может быть отрендерен](/docs/react-component.html#render), такой как элемент, строка или фрагмент. Следующий аргумент (`container`) — это DOM-элемент.

## Применение {#usage}

Обычно, когда вы возвращаете элемент из рендер-метода компонента, он монтируется в DOM как дочерний элемент ближайшего родительского узла:

```js{4,6}
render() {
  // React монтирует новый div и рендерит в него дочерние элементы
  return (
    <div>
      {this.props.children}
    </div>
  );
}
```

Но иногда требуется поместить потомка в другое место в DOM:

```js{6}
render() {
  // React *не* создаёт новый div. Он рендерит дочерние элементы в `domNode`.
  // `domNode` — это любой валидный DOM-узел, находящийся в любом месте в DOM.
  return ReactDOM.createPortal(
    this.props.children,
    domNode
  );
}
```

Типовой случай применения порталов — когда в родительском компоненте заданы стили `overflow: hidden` или `z-index`, но вам нужно чтобы дочерний элемент визуально выходил за рамки своего контейнера. Например, диалоги, всплывающие карточки и всплывающие подсказки.

> Примечание:
>
> При работе с порталами, помните, что нужно уделить внимание [управлению фокусом при помощи клавиатуры](/docs/accessibility.html#programmatically-managing-focus).
>
> Для модальных диалогов, убедитесь, что любой пользователь будет способен взаимодействовать с ними, следуя [практикам разработки модальных окон WAI-ARIA](https://www.w3.org/TR/wai-aria-practices-1.1/#dialog_modal).

[**Попробовать на CodePen**](https://codepen.io/gaearon/pen/yzMaBd)

## Всплытие событий через порталы {#event-bubbling-through-portals}

Как уже было сказано, портал может находиться в любом месте DOM-дерева. Несмотря на это, во всех других аспектах он ведёт себя как обычный React-компонент. Такие возможности, как контекст, работают привычным образом, даже если потомок является порталом, поскольку сам портал всё ещё находится в *React-дереве*, несмотря на его расположение в *DOM-дереве*.

Так же работает и всплытие событий. Событие, сгенерированное изнутри портала, будет распространяться к родителям в содержащем *React-дереве*, даже если эти элементы не являются родительскими в *DOM-дереве*. Представим следующую HTML-структуру:

```html
<html>
  <body>
    <div id="app-root"></div>
    <div id="modal-root"></div>
  </body>
</html>
```

`Родительский` компонент в `#app-root` сможет поймать неперехваченное всплывающее событие из соседнего узла `#modal-root`.

```js{28-31,42-49,53,61-63,70-71,74}
// Это два соседних контейнера в DOM
const appRoot = document.getElementById('app-root');
const modalRoot = document.getElementById('modal-root');

class Modal extends React.Component {
  constructor(props) {
    super(props);
    this.el = document.createElement('div');
  }

  componentDidMount() {
    // Элемент портала добавляется в DOM-дерево после того, как
    // потомки компонента Modal будут смонтированы, это значит,
    // что потомки будут монтироваться на неприсоединённом DOM-узле.
    // Если дочерний компонент должен быть присоединён к DOM-дереву
    // сразу при подключении, например, для замеров DOM-узла,
    // или вызова в потомке 'autoFocus', добавьте в компонент Modal
    // состояние и рендерите потомков только тогда, когда
    // компонент Modal уже вставлен в DOM-дерево.
    modalRoot.appendChild(this.el);
  }

  componentWillUnmount() {
    modalRoot.removeChild(this.el);
  }

  render() {
    return ReactDOM.createPortal(
      this.props.children,
      this.el
    );
  }
}

class Parent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {clicks: 0};
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    // Эта функция будет вызвана при клике на кнопку в компоненте Child,
    // обновляя состояние компонента Parent, несмотря на то,
    // что кнопка не является прямым потомком в DOM.
    this.setState(state => ({
      clicks: state.clicks + 1
    }));
  }

  render() {
    return (
      <div onClick={this.handleClick}>
        <p>Количество кликов: {this.state.clicks}</p>
        <p>
          Откройте DevTools браузера,
          чтобы убедиться, что кнопка
          не является потомком блока div
          c обработчиком onClick.
        </p>
        <Modal>
          <Child />
        </Modal>
      </div>
    );
  }
}

function Child() {
  // Событие клика на этой кнопке будет всплывать вверх к родителю,
  // так как здесь не определён атрибут 'onClick' 
  return (
    <div className="modal">
      <button>Кликните</button>
    </div>
  );
}

const root = ReactDOM.createRoot(appRoot);
root.render(<Parent />);
```

[**Попробовать на CodePen**](https://codepen.io/gaearon/pen/jGBWpE)

Перехват событий, всплывающих от портала к родительскому компоненту, позволяет создавать абстракции, которые не спроектированы специально под порталы. Например, вы отрендерили компонент `<Modal />`. Тогда его события могут быть перехвачены родительским компонентом, вне зависимости от того, был ли `<Modal />` реализован с использованием порталов или без них.
