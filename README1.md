# Actor model

Worker在mapbox以及web地图中占据相当重要的角色。因为，为了不影响主线性（UI）的性能，大量的计算和请求都放在worker中执行。

以下内容时探究mapbox是如何使用worker的，以及worker和主线程之间是如何工作的。

在开始之前，需要先补充一个设计模式，*参与者模式（Actor model）*。它是一种并发计算的模型，每个参与者都是一个基本的并发计算基本单元。

Mapbox通过创建参与者模型来实例化。每个参与者都具有几个属性和方法。

- 属性：
    1. target：表示拥有这个参与者的worker或者主线程
    2. Parent：当前实例的上级（不一定是主线程，也可能是worker）
    3. mapId：可选的参数，一般不传。在当前地图实例范围内，是一个唯一的标识符

- 方法：
    1. send：发送信息到主线程或者worker
    2. receive：接受信息的处理方法
    3. remove：移除监听`message`

Actor实例化的时候，在构造函数中会执行监听`message`事件，所以没有单独的监听方法，只有移除监听方法（remove）。

那么，他们是通过什么格式的信息来进行通信的呢？

主线程和worker之间通信都是通过`postMessage`和监听`message`事件实现的，因此，必须制定一个统一的通信格式标准。

`postMessage`方法只有worker或者主线程，所以，actor是调用了target对象上的方法。

mapbox的actor定义了一套数据格式：
```javascript
{
    targetMapId: targetMapId,
    sourceMapId: this.mapId,
    type: type,
    id: String(id),
    data: data
}
```
在收到了数据的时候，首先会比对`targetMapId`和`sourceMapId`是否不同，不同说明是来自其他worker或者主线程发送的，可以接收处理。

然后根据`type`的类型进行处理，`type`分为几种判断条件：
1. 是否是`<response>`，是的话，表示这是对一个事情的回复，可以调用`callback`函数了。
2. id存在，并且`type`是`parent`的一个方法，例如：`'loadTile'`,`'removeTile'`，那么就用`parent`来执行（通过传入`sourceMapId`，`data`，`done`）
3. 判断是否worker的上级是否有`getWorkSource`函数，这个函数返回当前worker可以做的事情的类型(WorkerSource)，包含各种类型（矢量、GeoJSON等）的`workerSource`，`workerSource`包含了瓦片的下载、重载瓦片、终止载入瓦片、移除瓦片、重新覆盖瓦片这几个函数，然后将执行`workerSource`方法中的一种。因此，在这种情况下，data.type是一个'{sourceType}.{method}'的样式，有两个标识符，`sourceType`表示`workerSource`的类型，例如workerSource的种类（矢量？还是GeoJSON），`method`表示`workerSource`的实例方法（loadTile？还是removeTile？）
4. 如果前面都没有命中，那么意味着只是想调用worker上级的方法。

因此，其实每个worker不知道自己是不是上级，因为自己可能有上级，也可能没有上级，这种复杂的逻辑结构，就需要有严格的静态类型检查，否则写着写着就不知道传来传去的数据是什么鬼了，因此就有了`WorkerSource`类型，由此衍生出了各种不同的这个类型的实例（矢量的workerSource或者GeoJSON的workerSource类型），他们都具有同样的方法名、参数和类似的方法作用，只是在具体方法的实现上略有不同。

由此可见，在阅读源码的时候，充分理解代码中定义的类型，有助于理清其脉络。
