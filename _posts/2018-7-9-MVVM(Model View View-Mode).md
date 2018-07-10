# MVVM(Model View View-Mode)

MVVM衍生于MVC，是对 MVC 的一种演进，它促进了 UI 代码与业务逻辑的分离。它正式规范了视图和控制器紧耦合的性质，并引入新的组件。他们之间的结构关系如下：

<img src="https://raw.githubusercontent.com/rannger/rannger.github.io/master/img/1874977-0fb12f6848ba6e78.png"/>

- MVVM的基本概念
    - 在MVVM 中，view 和 view controller正式联系在一起，我们把它们视为一个组件
    - view 和 view controller 都不能直接引用model，而是引用视图模型（viewModel）
    - viewModel 是一个放置用户输入验证逻辑，视图显示逻辑，发起网络请求和其他代码的地方
    - 使用MVVM会轻微的增加代码量，但总体上减少了代码的复杂性
- MVVM的注意事项
    - view 引用viewModel ，但反过来不行（即不要在viewModel中引入#import UIKit.h，任何视图本身的引用都不应该放在viewModel中）（PS：基本要求，必须满足）
    - viewModel 引用model，但反过来不行
- MVVM的使用建议 
    - MVVM 可以兼容你当下使用的MVC架构。
    - MVVM 增加你的应用的可测试性。
    - MVVM 配合一个绑定机制效果最好（PS：ReactiveCocoa你值得拥有）。
    - viewController 尽量不涉及业务逻辑，让 viewModel 去做这些事情。
    - viewController 只是一个中间人，接收 view 的事件、调用  viewModel 的方法、响应 viewModel 的变化。
    - viewModel 绝对不能包含视图 view（UIKit.h），不然就跟 view 产生了耦合，不方便复用和测试。
    - viewModel之间可以有依赖。
    - viewModel避免过于臃肿，否则重蹈Controller的覆辙，变得难以维护。
- MVVM的优势
    - 低耦合：View 可以独立于Model变化和修改，一个 viewModel 可以绑定到不同的 View 上
    - 可重用性：可以把一些视图逻辑放在一个 viewModel里面，让很多 view 重用这段视图逻辑
    - 独立开发：开发人员可以专注于业务逻辑和数据的开发 viewModel，设计人员可以专注于页面设计
    - 可测试：通常界面是比较难于测试的，而 MVVM 模式可以针对 viewModel来进行测试

- MVVM 的弊端
    - 数据绑定使得Bug 很难被调试。你看到界面异常了，有可能是你 View 的代码有 Bug，也可能是 Model 的代码有问题。数据绑定使得一个位置的 Bug 被快速传递到别的位置，要定位原始出问题的地方就变得不那么容易了。
    - 对于过大的项目，数据绑定和数据转化需要花费更多的内存（成本）。主要成本在于：
        - 数组内容的转化成本较高：数组里面每项都要转化成Item对象，如果Item对象中还有类似数组，就很头疼。
        - 转化之后的数据在大部分情况是不能直接被展示的，为了能够被展示，还需要第二次转化。
        - 只有在API返回的数据高度标准化时，这些对象原型（Item）的可复用程度才高，否则容易出现类型爆炸，提高维护成本。
        - 调试时通过对象原型查看数据内容不如直接通过NSDictionary/NSArray直观。
        - 同一API的数据被不同View展示时，难以控制数据转化的代码，它们有可能会散落在任何需要的地方。