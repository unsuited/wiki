##setState的理解
都说setState是异步的，那什么时候会是同步的呢，先说结论
* 在组件生命周期中或者react代理事件中，setState是异步更新的
* 在延时的回调或者js原生事件[例如addEventListener]中不一定是异步的

以后setState是否完全异步未知
#####在组件生命周期中或者react代理事件中，setState是异步更新的

例如
```
// example
// state.index 当前为 0
componentDidMount(){
    this.setState({index: this.state.index + 1});
    console.log(this.state.index); // 0
}
```

```
class Test extends Component {
  state = { counter: 0 };

  render() {
    return <div onClick={this.onClick}>点击</div>;
  }

  componentDidMount() {
    //手动绑定mousedown事件
    ReactDom.findDOMNode(this).addEventListener(
      "mousedown",
      this.onClick.bind(this)
    );

    //延时调用onclick事件
    setTimeout(this.onClick, 1000);
  }
  onClick = (event) => {
    if (event) {
      console.log(event.type);
    } else {
      console.log("timeout");
    }
    console.log("prev state:", this.state.counter);
    this.setState({
      counter: this.state.counter + 1
    });
    console.log("next state:", this.state.counter);
  };
}
export default Test;
```
使用三种方法更新state
- 在Component中绑定react代理的onClick事件
- 在componentDidMount中手动绑定原生js的mousedown事件
- 在componentDidMount中使用setTimeout调用onClick函数

输出的结果是
```
timeout
prev state: 0
next state: 1
mousedown
prev state: 1
next state: 2
click
prev state: 2
next state: 2
```
简单来说就是
```
// example
state = {
    index = 0
};
componentDidMount() {
   this.setState({ index: this.state.index + 1 );
   this.setState({ index: this.state.index + 1 );
}
// react中会认为是
Object.assign(
  previousState,
  {index: state.index+ 1},
  {index: state.index+ 1},
  ...
)
```
#####在延时的回调[setTimeout]或者js原生事件[例如addEventListener]中不一定是异步的
```
// example
state = {
    index = 0
};
render() {
    return <div onClick={this.onClick}>点击</div>;
}
componentDidMount() {
    setTimeout(() => {
      this.setState({ index: this.state.index + 1 );
      console.log(this.state.index); // 1
      this.setState({ index: this.state.index + 1 );
      console.log(this.state.index); // 2
    });
}

onClick = async e => {
    // react代理的事件中使用await也不一定是异步
    await this.setState({ index: this.state.index + 1 );
    console.log(this.state.index); // 3
    this.setState({ index: this.state.index + 1 );
    console.log(this.state.index); // 4
}
```
setState的异步过程【还没有捋清】
```
Component.prototype.setState = function (partialState, callback) {
  !(typeof partialState === 'object' || typeof partialState === 'function' || partialState == null) ? invariant(false, 'setState(...): takes an object of state variables to update or a function which returns an object of state variables.') : void 0;
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```
在setState中调用了enqueueSetState方法将传入的state放到一个队列中，接下来，看下enqueueSetState的具体实现：
```
  enqueueSetState: function (inst, payload, callback) {
    var fiber = get(inst);
    var currentTime = recalculateCurrentTime();
    var expirationTime = computeExpirationForFiber(currentTime, fiber);

    var update = createUpdate(expirationTime);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      {
        warnOnInvalidCallback$1(callback, 'setState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update, expirationTime);
    scheduleWork$1(fiber, expirationTime);
  },
```
