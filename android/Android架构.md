android 架构 MVC 、MVP、 MVVM 对比 - 爱码网

转自 Android 中的 MVC、MVP、MVVM 一、 MVC MVC 的全称是 Model-View-Controller，也就是模型 - 视图 - 控制器。

转自 [Android 中的 MVC、MVP、MVVM](https://www.jianshu.com/p/a8843574faf1)

## 一、 MVC

MVC 的全称是 Model-View-Controller，也就是模型 - 视图 - 控制器。

View 层：一般由 XML 布局文件充当。
Model 层：一些数据处理的工作，比如网络数据请求、数据库操作等。
Controller 层：通常由 Activity、Fragment 充当，并在其中进行界面、数据相关的业务处理。

可见在 Android 中，作为 Controller 的 Activity 或 Fragment 主要起到的作用就是解耦，将 View 和 Model 进行分离，两者在 Controller 中完成具体的操作。

下图是 MVC 架构的结构模型：

![img](https://www.likecs.com/default/index/img?u=aHR0cHM6Ly9waWFuc2hlbi5jb20vaW1hZ2VzLzI3OS81YzgxM2UyNGRhMDZmZWNhN2NjNDI0MDI0MzViZjVjNy5wbmc=)





## MVC 缺点：

\1. **耦合性高**：MVC 的耦合性还是相对较高的，Activity 或 Fragment 并非标准的 Controller，它一方面用来控制了布局，另一方面还要在 Activity 中写业务逻辑代码，造成了 Activity 或 Fragment 既像 View 又像 Controller，View 和 Model 之间可以互相访问，从而三者间构成回路。

2.**Controller 臃肿**：核心的业务逻辑都写在 Controller 中，导致 Controller 中的代码略显臃肿，这也是我们开发中使用 MVC 模式所面对的问题，Activity 或 Fragment 中的业务逻辑代码代码少则几百行多则上千行。

## 二、MVP

MVP 是 MVC 架构的一个演化版，全称是 Model-View-Presenter。

和 MVC 架构相比，MVP 架构在 Android 中场景应该是这样的:

Model 层：同样提供数据操作的功能

View 层：由 Activity、Fragment、或某个 View 控件担任

Presener 层：作为 View 层和 Model 层沟通的中介。

通常在 View 层中，有一个 Presenter 成员变量，同时我们的 View（可以是 Activity、Fragment）需要实现一个接口，这样可以将 View 层中需要的业务逻辑操作通过该接口转交给 Presenter 来实现，进而 Presenter 通过 Model 得到相应的数据，并通过 View 层实现的接口返回到 View 中。这样 View 层和 Model 层的耦合就解除了，同时也将业务逻辑从 View 中抽离出来，转移到 Presenter 中，避免了 Activity 或 Fragment 过度臃肿，充斥大量业务逻辑代码。

MVP 的结构模型如下：

![img](https://www.likecs.com/default/index/img?u=aHR0cHM6Ly9waWFuc2hlbi5jb20vaW1hZ2VzLzU4My82NTk2YWI5ODQ0NWZiNzYxZTc3MGY0YTM1MmIyNzFkZi5wbmc=)


从图中可以看出，MVP 架构中，解除了 View 和 Model 间的耦合，使它们不能相互访问，核心的业务逻辑都集中在 Presenter 中。

随着产品的升级，UI 会添加新的设计，业务逻辑也会更改，在 MVC 架构中，我们需要在 View 层处理这些变化，可是面对成百上千的代码，也是挺烦的。使用 MVP 架构能很好的降低 View 的复杂性，将业务逻辑分离到 Prestener，它们之间通过接口通信，处理变化时只需要根据情况来扩展接口并在 Prestener 处理新的业务逻辑即可。

所以可以看出，MVP 架构还是相当优良的，对于相对大的项目，它能够很好的组织处理应用的结构，使应用更加易于扩展、更加灵活。如果应用相对简单的话，使用 MVP 架构就相对复杂麻烦，有点小题大做的感觉。

## MVC 和 MVP 的区别

1.MVC 中是允许 Model 和 View 直接进行交互的，而 MVP 中，Model 与 View 之间的交互由 Presenter 完成；
2.MVP 模式就是将 P 定义成一个存放接口方法的地方，然后在每个触发的事件中调用对应接口方法来处理，也就是将逻辑放进了 P 中，需要执行某些操作的时候调用 P 的方法就行了。

## MVP 的优缺点

**优点：**

\1. **低耦合**，实现了 Model 和 View 真正的完全分离，可以修改 View 而不影响 Model；
\2. **模块职责划分明显**，层次清晰；
\3. **可以修改视图而不影响数据模型**，；
4.**Presenter 可复用**，一个 Presenter 可以用于多个 View，而不需要更改 Presenter 的逻辑；
\5. 利于测试驱动开发，以前的 Android 开发是难以进行单元测试的；
View 可以进行组件化，在 MVP 当中，View 不依赖 Model。

**缺点：**

1.Presenter 中除了应用逻辑以外，还有大量的 View——>Model，Model——>View 的手动同步逻辑，造成 Presenter 比较笨重，维护起来会比较困难；
\2. 由于对视图的渲染放在了 Presenter 中，所以视图和 Presenter 的交互会过于频繁，. 如果 Presenter 过多地渲染了视图，往往会使得它与特定的视图的联系过于紧密，一旦视图需要变更，那么 Presenter 也需要变更了。

## 三、MVVM

MVVM 的结构模型如下：

![img](https://www.likecs.com/default/index/img?u=aHR0cHM6Ly9waWFuc2hlbi5jb20vaW1hZ2VzLzY4NS9jMGM3MjNhZTE0ZjAzNjMzM2Q5MzExOTg0YWRmYjkxNS5wbmc=)


可以发现，MVVM 与 MVP 的结构还是很相似的：
**Model 层：**
Model 层保存了业务数据与处理方法，提供最终所需数据
**View 层**
对应 Activity 以及 XML，但是比起 MVC 与 MVP 框架中的 View 层，更加简洁
**ViewModel：**
负责实现 View 与 Model 的交互，将两者分离

View 层和 viewModel 层通过 DataBinding 进行了双向绑定，所以 Model 数据的更改会通过 viewModel 表现在 View 上；反之, View 收到操作，通过 viewmodel 去 Model 获取数据，且 viewModel 不持有 View，这就使得视图和控制层之间的耦合程度进一步降低，关注点分离更为彻底，同时减轻了 Activity 的压力。

所以 ViewModel 就是 View 和 Model 之间沟通的桥梁，根据具体情况处理 View 或 Model 的变化，解决了 View 和 Model 之间的耦合问题，让操作变得更加灵活、方便。

## MVVM 和 MVP 区别

ViewModel 承担了 Presenter 中与 view 和 Model 交互的职责，与 MVP 模式不同的是，ViewModel 与 View 之间是通过 Databinding 实现的，而 Presenter 是持有 View 的对象，直接调用 View 中的一些接口方法来实现。

Viewodel 可以理解成是 View 的数据模型和 Presenter 的合体。它通过双向绑定 (松耦合) 解决了 MVP 中 Presenter 与 View 联系比较紧密的问题。

## MVVM 的优点和缺点

**优点**

MVVM 模式和 MVC 模式一样，主要目的是分离视图（View）和模型（Model），有几大优点：

\1. **低耦合**：视图（View）可以独立于 Model 变化和修改，一个 ViewModel 可以绑定到不同的 View 上，当 View 变化的时候 Model 可以不变，当 Model 变化的时候 View 也可以不变。

2.**viewModel 可复用**：你可以把一些相似的逻辑放在一个 **viewModel** 里面，让很多相似功能的 view 都去绑定这同一个 viewModel, 重用这段视图逻辑。

3.**ViewModel 中解决了 MVP 中 V-P 互相持有引用的问题**

\4. **独立开发**：可以修改视图而不影响数据模型，开发人员可以专注于业务逻辑和数据的开发（ViewModel），设计人员可以专注于页面设计。
\5. **可测试**：界面素来是比较难于测试的，而现在测试可以针对 ViewModel 来写

**缺点**

\1. 数据绑定使得 Bug 很难被调试。你看到界面显示异常（不是崩溃）了，有可能是你 View 的代码有 Bug，也可能是 viewModel 的代码有问题。数据绑定使得一个位置的 Bug 被快速传递到别的位置，要定位原始出问题的地方就变得不那么容易了。

\2. 一个大的模块中，viewModel 也会很大，并且 viewModel 持有 Model 的依赖，长期持有，不释放内存，就造成了花费更多的内存。

\3. 数据双向绑定不利于 View 层代码重用。客户端开发最常用的重用是 view 布局，但是数据双向绑定技术，让你在一个 View 都绑定了一个 viewModel，不同模块的 viewModel 都不同。那就不能简单重用 View 了。

## MVI

我们使用 ViewModel 来承载 MVI 的 Model 层，总体结构也与 MVVM 类似, 主要区别在于 Model 与 View 层交互的部分

Model 层承载 UI 状态，并暴露出 ViewState 供 View 订阅，ViewState 是个 data class, 包含所有页面状态 View 层通过 Action 更新 ViewState，替代 MVVM 通过调用 ViewModel 方法交互的方式 MVI 实例介绍

添加 ViewState 与 ViewEvent

ViewState 承载页面的所有状态，ViewEvent 则是一次性事件，如 Toast 等, 如下所示

dataclassMainViewState(val fetchStatus: FetchStatus, val newsList: List<NewsItem>) sealedclassMainViewEvent{dataclassShowSnackbar(val message: String) : MainViewEvent()dataclassShowToast(val message: String) : MainViewEvent()} 我们这里 ViewState 只定义了两个，一个是请求状态，一个是页面数据 ViewEvent 也很简单，一个简单的密封类，显示 Toast 与 SnackbarViewState 更新

classMainViewModel : ViewModel() {privateval _viewStates: MutableLiveData<MainViewState> = MutableLiveData()val viewStates = _viewStates.asLiveData()privateval _viewEvents: SingleLiveEvent<MainViewEvent> = SingleLiveEvent()val viewEvents = _viewEvents.asLiveData()init { emit(MainViewState(fetchStatus = FetchStatus.NotFetched, newsList = emptyList())) }privatefunfabClicked() { count++ emit(MainViewEvent.ShowToast(message = "Fab clicked count $count")) }privatefunemit(state: MainViewState?) { _viewStates.value = state }privatefunemit(event: MainViewEvent?) { _viewEvents.value = event }} 如上所示

我们只需定义 ViewState 与 ViewEvent 两个 State, 后续增加状态时在 data class 中添加即可，不需要再写模板代码 ViewEvents 是一次性的，通过 SingleLiveEvent 实现，当然你也可以用 Channel 当来实现当状态更新时，通过 emit 来更新状态 View 监听 ViewState

privatefuninitViewModel() { viewModel.viewStates.observe(this) { renderViewState(it) } viewModel.viewEvents.observe(this) { renderViewEvent(it) } } 如上所示，MVI 使用 ViewState 对 State 集中管理，只需要订阅一个 ViewState 便可获取页面的所有状态，相对 MVVM 减少了不少模板代码。

View 通过 Action 更新 State

classMainActivity : AppCompatActivity() {privatefuninitView() { fabStar.setOnClickListener { viewModel.dispatch(MainViewAction.FabClicked) } }}classMainViewModel : ViewModel() {fundispatch(action: MainViewAction) = reduce(viewStates.value, action)privatefunreduce(state: MainViewState?, viewAction: MainViewAction) {when (viewAction) {is MainViewAction.NewsItemClicked -> newsItemClicked(viewAction.newsItem) MainViewAction.FabClicked -> fabClicked() MainViewAction.OnSwipeRefresh -> fetchNews(state) MainViewAction.FetchNews -> fetchNews(state) } }} 如上所示，View 通过 Action 与 ViewModel 交互，通过 Action 通信，有利于 View 与 ViewModel 之间的进一步解耦，同时所有调用以 Action 的形式汇总到一处，也有利于对行为的集中分析和监控

它主要有以下优势

强调数据单向流动，很容易对状态变化进行跟踪和回溯使用 ViewState 对 State 集中管理，只需要订阅一个 ViewState 便可获取页面的所有状态，相对 MVVM 减少了不少模板代码 ViewModel 通过 ViewState 与 Action 通信，通过浏览 ViewState 和 Aciton 定义就可以理清 ViewModel 的职责，可以直接拿来作为接口文档使用。当然 MVI 也有一些缺点，比如

所有的操作最终都会转换成 State，所以当复杂页面的 State 容易膨胀 state 是不变的，因此每当 state 需要更新时都要创建新对象替代老对象，这会带来一定内存开销