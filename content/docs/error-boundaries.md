---
id: error-boundaries
title: Error Boundaries
permalink: docs/error-boundaries.html
---

Trước đây, các lỗi JavaScript bên trong component sẽ làm hỏng các state của React và [emit](https://github.com/facebook/react/issues/4026) [các error](https://github.com/facebook/react/issues/8579) [cryptic](https://github.com/facebook/react/issues/6895)  trong các lần render tiếp theo. Lỗi xuất hiện trong code ứng dụng trước đó là nguyên nhân, nhưng React chưa cung cấp một cách để xử lý trong component, và không thể phục hồi lại chúng.


## Giới thiệu Error Boundary {#introducing-error-boundaries}

Một lỗi Javascript bên trong UI không nên làm chết cả ứng dụng. Để giải quyết vấn đề này cho người dùng React, React 16 giới thiệu một khái niệm mới "error boundary".

Các Error boundary là những React component dùng để **bắt lỗi JavaScript ở bất kỳ đâu bên trong các component con, log các error này, và hiển thị một UI thay thế** thay vì làm hư cả một cây component. Các error boundary bắt lại lỗi trong quá trình render, trong các phương thức lifecycle, và trong constructor của cả cây bên dưới chúng.

> Lưu ý
>
> Error boundary **không** bắt lỗi cho:
>
> * Event handler ([xem thêm](#how-about-event-handlers))
> * Code Async (ví dụ. `setTimeout` hoặc `requestAnimationFrame`)
> * Server side rendering
> * Error được đẩy ra từ chính error boundary (thay vì là con của nó)

Một class component trở thành error boundary nếu nó có khai báo phương thức lifecycle [`static getDerivedStateFromError()`](/docs/react-component.html#static-getderivedstatefromerror) hoặc [`componentDidCatch()`](/docs/react-component.html#componentdidcatch). Sử dụng `static getDerivedStateFromError()` để render một UI thay thế sau khi có lỗi xuất hiện. Sử dụng `componentDidCatch()` để log thông tin lỗi.

```js{7-10,12-15,18-21}
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Cập nhập state để lần render sẽ hiển fallback UI.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Bạn cũng có thể log error vào một dịch vụ ghi nhận error
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // Bạn có thể render bất kỳ fallback UI tùy biến nào
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children; 
  }
}
```

Sau đó bạn có thể sử dụng nó như một component bình thường:

```js
<ErrorBoundary>
  <MyWidget />
</ErrorBoundary>
```

Error boundary làm việc giống như JavaScript `catch {}`, nhưng dành cho component. Error boundary *phải* là một class component. Trong thực tế, hầu hết thời gian chúng ta chỉ cần khai báo một component error boundary một lần và sử dụng nó trong toàn bộ ứng dụng.

Lưu ý **error boundary chỉ bắt lỗi trong component bên dưới**. Một error boundary không thể bắt một error bên trong nó. Nếu một error boundary thất bại trong việc render thông báo lỗi, lỗi sẽ được đẩy lên error boundary gần nhất. Giống như cách làm việc của `catch {}` trong JavaScript.


## Xem demo thực tế {#live-demo}

Xem [ví dụ khai báo và sử dụng error boundary](https://codepen.io/gaearon/pen/wqvxGa?editors=0010) với [React 16](/blog/2017/09/26/react-v16.0.html).


## Đặt Error Boundary ở đâu {#where-to-place-error-boundaries}

Độ chi tiết của error boundary tùy thuộc vào bạn. Bạn có thể bọc route component trên cùng để hiển thị thông báo "Something went wrong", giống như các server-side framework xử lý khi có lỗi. Bạn cũng có thể bọc các component độc lập trong một error boundary để tránh việc làm hư toàn bộ ứng dụng.


## Behavior mới cho các lỗi chưa xử lý{#new-behavior-for-uncaught-errors}

Thay đổi mang một hàm ý rất quan trọng. **Từ React 16, lỗi nếu không được bắt bởi bất cứ error boundary nào sẽ dẫn đến kết quả unmount toàn bộ cây component React**

Chúng tôi đã tranh cãi trên quyết định này, tuy nhiên theo kinh nghiệm thực tế việc tệ nhất là cứ để một UI bị lỗi xuất hiện, thà là xóa hẳn nó đi. Ví dụ, trong sản phẩm như Messenger để một UI bị lỗi xuất hiện có thể dẫn đến việc ai đó gửi đi một tin nhắn đến nhầm người. Tương tự, sẽ rất tệ cho các ứng dụng thanh toán khi hiển thị sai số tiền thay vì không hiển thị gì cả.

Thay đổi này nghĩa là khi bạn chuyển sang dùng React 16, các trường hợp bị lỗi bạn chưa xử lý, nhưng đã tồn tại sẽ cần được tính đến. Thêm error boundary cho phép bạn cung cấp trải nghiệm người dùng tốt hơn khi có gì đó sai.

Lấy ví dụ, Facebook Messenger bọc nội dung bên trong sidebar, bảng info, và hộp thoại nhập tin nhắn trong các error boundary riêng. Nếu một vài component trong một các UI đó lỗi, các phần khác vẫn hoạt động bình thường.

Chúng tôi khuyến khích sử dụng các dịch vụ thông báo JS error  (hoặc tự làm) để biết được khi nào có lỗi trong lúc sử dụng và fix nó.

## Component Stack Trace {#component-stack-traces}

React 16 in toàn bộ lỗi xuất hiện lúc render vào console lúc phát triển, thậm chí nếu ứng dụng vô tình quăng ra. Thêm nữa với thông báo lỗi và JavaScript stack, nó sẽ cung cấp component stack trace. Bạn có thể xem chính xác component bị lỗi ở đâu trong cây component:

<img src="../images/docs/error-boundaries-stack-trace.png" style="max-width:100%" alt="Error caught by Error Boundary component">

Bạn cũng có thể xem tên file và dòng bị lỗi trong component stack trace. Chạy mặc định trong [Create React App](https://github.com/facebookincubator/create-react-app):

<img src="../images/docs/error-boundaries-stack-trace-line-numbers.png" style="max-width:100%" alt="Error caught by Error Boundary component with line numbers">

Nếu bạn không sử dụng Create React App, bạn có thể thêm [plugin này](https://www.npmjs.com/package/babel-plugin-transform-react-jsx-source) thủ công với cấu hình Babel. Lưu ý nó chỉ được dùng cho môi trường development và **phải disable trong production**.

> Lưu ý
>
> Tên Component được hiển thị bên trong stack trace tùy thuộc vào [`Function.name`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/name). Nếu bạn hỗ trợ các trình duyệt cũ hơn và thiết bị không hỗ trợ từ đầu (như IE11), nên thêm một polyfill `Function.name` trong ứng dụng, ví dụ như [`function.name-polyfill`](https://github.com/JamesMGreene/Function.name). Hoặc là, bạn có thể rõ ràng đặt [`displayName`](/docs/react-component.html#displayname) trên tất cả các component.


## Vậy còn try/catch? {#how-about-trycatch}

`try` / `catch` rất tốt nhưng chỉ làm việc khi viết theo hướng chỉ định rõ từng bước cần làm:

```js
try {
  showButton();
} catch (error) {
  // ...
}
```

Tuy nhiên, React component theo hướng quan trọng là kết quả nhận được và chỉ rõ *cái gì* nên render:

```js
<Button />
```

Error boundary giữ được các viết trong React, và hoạt động như chúng ta mong đợi. Lấy ví dụ, thậm chí nếu một lỗi xuất hiện trong phương thức `componentDidUpdate` gây ra bởi `setState` đâu đó bên trong cây component, nó vẫn đẩy lên trên error boundary gần nhất.

## Vậy còn Event Handler? {#how-about-event-handlers}

Error boundary **không** bắt error trong event handler.

React không cần error boundary để khôi phục bên trong event handler. Không giống như phương thức render và phương thức lifecycle, event handler không xảy ra trong lúc render. Khi chúng được đẩy ra, React vẫn biết sẽ hiển thị gì trên màn hình.

Bạn cần catch một error bên trong event handler, sử dụng câu lệnh `try` / `catch` Javascript thông thường:

```js{9-13,17-20}
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = { error: null };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    try {
      // làm gì đó có thể throw
    } catch (error) {
      this.setState({ error });
    }
  }

  render() {
    if (this.state.error) {
      return <h1>Caught an error.</h1>
    }
    return <div onClick={this.handleClick}>Click Me</div>
  }
}
```

Ở ví dụ trên mô tả một cách chạy bình thường của JavaScript và không sử dụng Error Boundary.

## Cách đặt tên thay đổi từ React 15 {#naming-changes-from-react-15}

React 15 chỉ gồm một phần hổ trợ rất nhỏ cho error boundary bằng một phương thức khác `unstable_handleError`. Phương thức này sẽ không còn chạy, và bạn sẽ cần đổi nó thành `componentDidCatch` trong code bắt đầu từ bản 16 beta được công bố.

Với sự thay đổi này, chúng tôi cung cấp [codemod](https://github.com/reactjs/react-codemod#error-boundaries) để tự động cập nhập code của bạn.
