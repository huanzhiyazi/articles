<a name="index">**目录**</a>

- <a href="#ch1">**1 Data Binding 的结构**</a>
    * <a href="#ch1.1">1.1 rebind 行为</a>
    * <a href="#ch1.2">1.2 observe data 行为</a>
    * <a href="#ch1.3">1.3 observe view 行为</a>

<br>
<br>

### <a name="ch1">1 Data Binding 的结构</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

作为在 Android 开发中体现 MVVM 架构思想的 Data Binding，其核心是 **观察者模式** 的特定实现。首先，它有三个主要的实体：

1. **Data**：与 View 相关的数据，它可以是 View 的可观察者对象；
2. **View**：展示给用户的视图，如果有交互功能且能更新数据，它可以是 Data 的可观察者对象；
3. **ViewDataBinding**：连接 Data 和 View 的中介，当 Data 或 View 作为可观察者对象时，它充当可观察者对象的代理。假如当我们写了一个名为 demo.xml 的 Data Binding 的 layout 文件后，编译工具会生成一个相应的类——DemoBinding，它的原型就是 ViewDataBinding。我们通常通过 ```DataBindingUtil.inflate(inflater, R.layout.demo, container, false)``` 来实例化的 DemoBinding 对象，即 ViewDataBinding。

其次，它主要提供了以下三个方面的功能：

1. 将特定的 View 与特定的 Data 进行绑定，便于模块化；
2. View 自动感知和响应 Data 的变化，使得处理数据的业务层不必关心 View 的状态，便于解耦；
3. Data 也可以自动同步带有交互功能的 View 对数据的修改，使得 UI 层的交互不必担心数据是否能同步 View 状态的问题，仍然便于解耦。

基于这三个功能，Data Binding 的结构也可以划分为三个行为模式，以下一一介绍：

#### <a name="ch1.1">1.1 rebind 行为</a>

首先，Data 往往是一个数据的集合，数据绑定的第一步就是要将整个 Data 集合绑定到 View，比如初始化和数据的整体更新，如下图所示：

![Data Binding rebind](images/databinding_rebind.png "Data Binding rebind")

可以观察到，rebind 的过程就是一个简单的赋值操作，将 View 的值设置为 Data，只不过由 ViewDataBinding 这个代理来完成这个工作。图中的 ```_all``` 参数表示将 View 的所有需要更新的节点都设置为 Data 的所有对应的成员值。

#### <a name="ch1.2">1.2 observe data 行为</a>

有时候，我们并不需要每次更新整个 Data 集合，而只需要更新集合中的某一个成员。我们希望看到的结果是，当 Data.element_i 发生变化的时候，View.child_i 更新就可以了，而不需要将 View 的所有视图节点都重新渲染一遍。要做到这一点，我们必须要让 View 可以观察 Data 的行为。换句话说，Data 是一个可观察者对象——这是 Data Binding 中另一个魅力所在，其行为模式如下：

![Data Binding observe data](images/databinding_observe_data.png "Data Binding observer data")

我们可以将任何数据作为一个 Observable，然后将 ViewDataBinding 作为 View 的代理观察者，订阅 Data 的成员变化，一旦 Data 成员变化，便通知所有观察者对象——即 ViewDataBinding，然后 ViewDataBinding 再将 View 的相应节点的值设置为 Data 相应成员的新值——即图中的 ```_member``` 参数。

在 Data Binding 框架中，将 Data 设置为 Observable 的方式类似于下面的代码：

```java
public class DemoData extends BaseObservable {
    private int element;

    // 定义其它成员，省略

    @Bindable
    public int getElement() {
        return this.element;
    }

    public void setElement(int e) {
        this.element = e;
        notifyPropertyChanged(BR.element);
    }

    // 省略其它成员操作
}
```

这里有三个关键部分：

1. **BaseObservable**：可观察者基类（实际的祖先基类是一个 Observable 接口），实现改接口后，ViewDataBinding 就会在每次 rebind 的时候去订阅 Data 的变化；
2. **@Bindable 标注**：声明该成员是可被观察的，以及在 layout 中可以以 Data.xxx（标注的方法名如果为 getXXX）的方式进行访问；
3. **notifyPropertyChanged 方法**：BaseObservable 用于通知具体成员发送变化的方法，只要该方法被调用，ViewDataBinding 就会检索出是哪一个 element 的变化，并只对 View 相应的节点进行更新。

#### <a name="ch1.3">1.3 observe view 行为</a>

在开发中，根据业务需求，我们一般能遇到两种类型的 View：

- 一种是只用于展示的 View，它只展示 UI 状态，而不反馈状态，我们称之为 **单工View**；
- 另一种除了展示以外，还会反馈状态给监听者，我们称之为 **双工View**。
 
在 Android 的 UI事件流中，因为所有的 View 都是可以反馈状态的，所以准确来说，所有的 View 其实都是双工的。我们在这里区分单工和双工是针对业务需求的，比如：我们很多的视图只需要它们展示就可以了，不需要监听它们的状态变化，那么我们将其归为单工View。

费尽心思进行这样的划分，是因为，单工View 只需要有 observe data 行为就可以了；而双工View 往往就需要 observe view 行为。具体来说，在反馈状态时需要更新 Data 的双工 View，我们需要进行 observe view 行为。

因为双工View 会更新 Data，所以为了保证数据的一致性，Data 需要观察双工View 的状态变化。要做到这一点，这样的双工View 必须是一个可观察者对象。得益于 UI事件流的实现，双工View天然是可观察的，在自定义的双工View中，可以间接引用 ViewDataBinding，这样 ViewDataBinding 就可以代理 Data 订阅 View 的状态变化：

![Data Binding observe view](images/databinding_observe_view.png "Data Binding observe view")




