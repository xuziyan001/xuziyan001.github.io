---
layout: post
title:  "Design Patterns(3)"
date:   2014-08-20 11:20:00
categories: Alibaba-inc update
---
<p>
行为模式一共细分了11个类别，它们涉及到算法和对象间职责的分配，有些模式可能难以在简短的代码层面给出一个很好地诠释，所以只能尽量的去理解它们。看完设计模式之后，估计还需要去看看《大话设计模式》去接接地气。
</p>
<h3>Chain of Responsibility(职责链)</h3>
<p>
沿着一条职责链传递请求，直到有一个对象处理它为止，从而避免请求的发送者和接受者之间的耦合关系。在实现上，每一个Handler维护一个后继者链接，并提供HandleRequest的缺省实现：HandleRequest向后继者(若有)转发请求。
</p>
{%highlight cpp%}
typedef int Topic;
const Topic NO_HELP_TOPIC = -1;
class HelpHandler{
public:
  HelpHandler(HelpHandler* =0,Topic=NO_HELP_TOPIC);
  virtual bool HasHelp();
  virtual void SetHandler(HelpHandler*,Topic);
  virtual void HandleHelp();
private:
  HelpHandler* _successor;
  Topic _topic;
};
HelpHandler::HelpHandler(HelpHandler* h, Topic t):_successor(h),_topic(t){}
bool HelpHandler::HasHelp(){return _topic!=NO_HELP_TOPIC;}
void HelpHandler::HandleHelp(){
  if (_successor != 0){
    _successor->HandleHelp()
  }
}
{%endhighlight%}
这里定义HelpHandler类处理帮助请求的接口关键操作是HandleHelp，它可以被子类重新定义；HasHelp用于检查是否有有一个相关的帮助主题。所有的窗口组件(Button,Dialog...)都是Widget的子类，而Widget是HelpHandler的子类：
{%highlight cpp%}
class Widget : public HelpHandler{
protected:
  Widget(Widget* parent, Topic t=NO_HELP_TOPIC);
private:
  Widget* _parent;
};
class Button : public Widget{
public:
  Button(Widget* d, Topic t=NO_HELP_TOPIC);
  virtual void HandleHelp();//Button重载的HandleHelp方法
};
void Button::HandleHelp(){
  if ( HasHelp() ){
    //提供Button的Help
  }else{
    HelpHandler::HelpHandler();
  }
}
{%endhighlight%}
Button首先检测自身是否有帮助主题，若没有则用HelpHandler中的HandleHelp操作转发给它的后继者。但下面的Dialog类实现了一个类似的策略，只不过它的后继者不是一个窗口组件，而是任意的帮助请求处理对象(这个后继是Application的一个实例)。
{%highlight cpp%}
class Dialog : public Widget{
public:
  Dialog(HelpHandler* h, Topic t=NO_HELP_TOPIC);//注意第一个参数是HelpHandler
  virtual void HandleHelp();//Button重载的HandleHelp方法
};
Dialog::Dialog(HelpHandler* h, Topic t):Widget(0){
  SetHandler(h,t);//指定到链的末端
}
void Dialog::HandleHelp(){
  if ( HasHelp() ){
    //提供Dialog的Help
  }else{
    HelpHandler::HelpHandler();
  }
}
class Application : public HelpHandler{//Application不是窗口组件
  Application(Topic t):HelpHandler(0,t){}
  virtual void HandleHelp(){
  //提供一系列的Help Topics
  }
};
{%endhighlight%}
Application是链的末端，当一个帮助请求传递到这一层时，该应用可提供关于该应用的一般性信息。一下代码创建并连接这些对象。
{%highlight cpp%}
const Topic PRINT_TOPIC = 1;
const Topic PAPER_ORIENTATION_TOPIC = 2;
const Topic APPLICATION_TOPIC = 3;
Application* application = new Application(APPLICATION_TOPIC);
Dialog* dialog = new Dialog(application,PRINT_TOPIC);
Button* button = new Button(dialog,PAPER_ORIENTATION_TOPIC);
{%endhighlight%}
这段代码看了三遍，终于是稍微明白了职责链的运行模式。因为职责链的后继关系和单纯的继承关系还不太一样，后继的层次可以很深，所以理解起来可能比较费力，但是只要知道了这种设计模式的要点，通过阅读相关代码和画图的方法，还是可以完全明白的。
<h3>Command(命令)</h3>
<p>
命令模式将一个请求封装为一个对象，从而可用不同的请求对客户进行参数化、对请求排队或记录请求日志以及支持可撤销的操作。命令的别名也叫事务(Transaction)，在数据库操作中使用得比较多。命令模式的关键是一个抽象的Command类，它定义了一个执行操作的接口，具体的Command子类将接受者作为其一个实例变量，并实现Execute操作。一下实现一个简单的不能取消和不需要参数的命令，用类模板来参数化该命令的接收者。
</p>
{%highlight cpp%}
class Command{//抽象的Command类
public:
  virtual ~Command();
  virtual void Excute()=0;
protected:
  Command();
};
template<class Receiver>
class SimpleCommand : public Command{
public:
  typedef void (Receiver::*Action)();
  SimpleCommand(Receiver* r, Action a):_receiver(r),_action(a){}
  virtual void Excute();
private:
  Action _action;
  Receiver* _receiver;
};
template <class Receiver>
void SimpleCommand<Receiver>::Excute(){
  (_receiver->*_action)();
}
{%endhighlight%}
这里使用了函数指针，阅读起来可能有点麻烦，但这也是C++炫酷的地方！为创建一个调用Myclass类的一个实例上的Action方法的Command对象，仅需调用一下代码：
{%highlight cpp%}
Myclass recerver = new Myclass;
//...
Command aCommand = new SimpleCommand<Myclass>(recerver,&Myclass::Action);
aCommand->Excute();
{%endhighlight%}
如果使用更复杂的命令，不仅要维护它们的接收者，而且还要登记参数，保存用于取消操作的状态，此时就需要专门定义一个Command的子类。
<h3>Interpreter(解释器)</h3>
<p>
给定一个语言，定义它的文法的一种表示，并定义一个解释器，该解释器使用该表示来解释语言中的句子，这个模式应该在各语言的解释器模块中被大量使用。之前还写过一个基于解释器模式的计算器，当时是在网上照葫芦画瓢，不是很理解，但是在计算器的实现上使用解释器似乎有点大材小用。对于一个语言编译器来说，解释器模式的使用似乎更加能够体现语言作者赋予该语言的思想。
</p>
<p>
此处就不再以代码的形式赘述解释器模式了，有兴趣的同学可以去看看gcc的源码，哈哈。解释器模式的设计中，要区分终结符表达式(TerminalExpression)和非终结符表达式(NonterminalExpression)，这个和组合(Composition)设计模式有些类似，只是解释器模式将行为抽象成抽象语法树中的节点。不仅如此，解释器模式还可以和享元模式(Flyweight)结合起来使用，在抽象语法树中共享终结符。
</p>
<h3>Iterator(迭代器)</h3>
<p>
迭代器是广为人知的一种设计模式之一，它提供一种方法顺序访问聚合对象中各个元素，而又不需暴露该对象的内部表示。在C++中，STL的迭代器结合了代理模式(Proxy)和迭代器模式的特点，代理总是在栈上分配资源，而其析构器负责删除真正的迭代器，并且代理内联重载了操作符“->”和“*”，使得它在使用起来与迭代器没有任何差别，并且不产生任何额外开销。
</p>
<p>
迭代器模式在C++中使用得尤其出神入化，这里推介<a href="http://jjhou.boolan.com/">侯捷</a>老师写的《C++源码剖析》, 当时是在三星通信研究院实习的时候，百无聊赖把这本书给笼统看了一遍，现在想想，里面涉及到了很多之前我们学过的设计模式，像生成器、抽象工厂、代理、迭代器等等，难怪侯捷老师把STL源码评为世界上写得最好的代码之一。要是之后又时间应该把《源码》这本书按照《设计模式》的形式重新细读一遍，体会体会代码写到极致是一种什么感觉！
</p>
<h3>Mediator(中介者)</h3>
<p>
中介模式比较好理解，它用一个中介对象来封装一系列对象的交互。中介模式对对象如何协作进行了抽象，将各同事类(Colleague)解耦，从而减少了子类的生成。与外观模式(Facade)不同的是，中介者提供了各Colleague对象不支持或不能支持的协作行为，它的协议是多向的；而外观对象对这个子系统类提出请求，但反之不行，它的协议是单向的。下面实现一个FontDialogDirectator类对在对话框中的窗口组件进行中介，它是抽象类DialogDirector的子类。
</p>
{%highlight cpp%}
class FontDialogDirector : public DialogDirector{
public:
  FontDialogDirector();
  virtual ~FontDialogDirector();
  virtual void WidgetChanged(Widget*);
protected:
  virtual void CreateWidgets();
private:
  Button* _ok;
  Button* _cancel;
  ListBox* _fontList;
  EntryField* _fontName;
};
{%endhighlight%}
FontDialogDirector跟踪它显示的窗口组件组件，它重定义了CreateWidgets以创建窗口组件并初始化对它们的应用。WidgetChanged保证窗口组件正确地协同工作。
{%highlight cpp%}
void FontDialogDirector::CreateWidgets(){
  _ok = new Button(this);
  _cancel = new Button(this);
  _fontList = new ListBox(this);
  _fontName = new EntryField(this);
  //assemble the widgets in the dialog
}
void FontDialogDirector::WidgetChanged(Widget* theChangedWidget){
  if(theChangedWidget == _fontList){
    _fontName->SetText(_fontList->GetSelection());
  }else if(theChangedWidget == _ok){
    //apply font change and dismiss dialog
  }else if(theChangedWidget == _cancel){
    //dismiss dialog
  }
}
{%endhighlight%}
由于WidgetChanged的复杂度随着对话框的复杂度增加而增加，所以在设计的时候要尽量避免大对话框的出现，以免中介者的复杂性抵消该模式在其他方面带来的好处。
<h3>Memento(备忘录)</h3>
<p>
在不破坏封装的情况下，捕获一个对象的内部状态，并且在该状态之外保存这个状态。此设计模式广泛应用于需要支持撤销操作的应用中，Execute在执行之前首先获取一个Memento备忘录，UnExcute先逆向执行Excute的动作，再将状态设回原先的状态。备忘录模式常常和命令模式(Command)结合使用，每个命令类里保有一个相关的Memento实例，它调用原发器(Originator)的实例创建。原发器即该备忘录中存储的状态所来源的对象，可以理解为备忘录保有的是原发器在某个时刻的状态，只有原发器可以向备忘录存取信息，备忘录对其他的对象不可见。在实现上，<b>备忘录类可以声明原发器是自己的私有友元</b>。
</p>
{%highlight cpp%}
class State;
class Originator{
public:
  Memento* CreateMemento();
  void SetMemento(const Memento*);
  //...
private:
  State* state;//internal data structures
  //...
};
class Memento{
public:
  virtual ~Memento();
  //narrow public interface
private:
  friend class Originator;
  Memento();
  //private menbers accessible only to Originator
{%endhighlight%}
<h3>Observer(观察者)</h3>
<p>
观察者类似于发布-订阅(Publish-Subscribe)系统，它定义对象间一对多的关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并自动更新。最早观察者模式应该是著名的MVC结构了，它现在也是很多语言的用户界面框架。
</p>
<p>
下面的代码中实现了两个抽象类Observer和它所观察的对象Subject，Observer的Update操作可以支持一个观察者观察多个目标，根据参数让目标判定是哪一个目标发生了变化。
</p>
{%highlight cpp%}
class Subject;
class Observer{
public:
  virtual ~Observer();
  virtual void Update(Subject* theChangedSubject) = 0;
protected:
  Observer();
};
//
class Subject{
public:
  virtual ~Subject();
  virtual void Attach(Observer*);
  virtual void Detach(Observer*);
  virtual void Notify();
protected:
  Subject();
private:
  List<Observers*> *_observers;
}
void Subject::Attach(Observer* o){
  _observers->Append(o);
}
void Subject::Detach(Observer* o){
  _observers->Remove(o);
}
void Subject::Notify(){
  ListIterator<Observer*> i(_observers);
  for(i.First();!i.IsDone();i.Next()){
    i.CurrentItem()->Update(this);
  }
}
{%endhighlight%}
假定ClockTimer是Subject的一个子类，用于存储和维护一天时间的具体目标，它的Tick函数每一秒钟通知一次它的观测者，现在我们就可以定义DigitalClock和AnalogClock类来显示时间，它们都是Widget的子类。
{%highlight cpp%}
class DigitalClock : public Widget,public Observer{
public:
  DigitalClock(ClockTimer*);
  virtual ~DigitalClock();
  virtual void Update(Subject*);
  virtual void Draw();//override Widget operation
private:
  ClockTimer* _subject;
};
DigitalClock::DigitalClock(ClockTimer* s){
  _subject = s;
  _subject->Attach(this);
}
DigitalClock::~DigitalClock(){
  _subject->Detach(this);
}
void DigitalClock::Update(Subject* theChangedSubject){
  if(theChangedSubject == _subject){//check Subject before Update
    Draw();
  }
}
{%endhighlight%}
AnalogClock可以用类似的方法构建，下面的代码创建一个AnalogClock和一个DigitalClock，它们总是显示相同的时间：
{%highlight cpp%}
ClockTimer* timer = new ClockTimer;
AnalogClock* analogClock = new AnalogClock(timer);
DigitalClock* digitalClock = new DigitalClock(timer);
{%endhighlight%}
一旦timer走动，两个时钟都会被更新并正确地重新显示。
<h3>State(状态)</h3>
<p>
状态，亦可以称为状态对象，它允许一个对象在其内部状态改变时改变它的行为，就好像这个对象改变了这个类一样。实际上，状态模式将不同状态的行为分割开来，使得状态转换显式化。程序的Context不知道所有状态的转移过程，只是在构造函数中将状态对象初始化，并提供一个ChangeState操作来改变自身的状态。State类被声明为Context的友元，从而给了它访问这一操作的特权。
</p>
{%highlight cpp%}
void State::ChangeState(Context* c, State* s){
  c->ChangeState(s);
}
{%endhighlight%}
State内部交互有点向有限状态机，并且State子类没有局部状态，因此它们可以被共享，并且每个子类只需一个实例，它可以由静态的Instance操作得到。在完成于状态相关的工作后，这些操作调用ChangeState操作来改变Context的状态。
{%highlight cpp%}
void ConcreteStateA::ActionA(Context* c){
  //do things
  ChangeState(c, ConcreteStateB::Instance());
}
{%endhighlight%}
当一个操作中含有庞大的多分支的条件语句，且这些分支依赖于该对象的状态，State模式可以将每一个条件分支放入一个独立的类中，该类的对象可以不依赖于其他对象而独立变化。
<h3>Strategy(策略)</h3>
<p>
策略模式是封装一系列的算法，使得它们可以相互替换，并且独立于使用它的客户而变化。这个模式有点像装饰模式(Decorator)的类型，分离了Strategy和Context，可以使具体的Strategy的子类来修饰同一个Context。我们可以利用模板机制用一个Strategy来配置一个类，它将Strategy和它的Context静态地绑定在一起，从而提高了效率。
</p>
{%highlight cpp%}
template <class AStrategy>
class Context{
  void Operation(){theStrategy.DoAlgorithm();}
  //...
private:
  AStrategy theStrategy;
}
{%endhighlight%}
当它被实例化时用一个具体的Strategy类来配置：
{%highlight cpp%}
template <class AStrategy>
class MyStrategy{
public:
  void DoAlgorithm();
};
Context<MyStrategy> aContext;
{%endhighlight%}
当然，一个Strategy模式的设计不会这么简单，还有很多需要考虑的细节，比如Context的数据接口需要以何种粒度编写来控制与Strategy的耦合程度；是否需要对Strategy定义缺省行为等，并且，在Strategy模式下，客户需要了解每个Strategy，这就可能不得不向客户暴露具体的实现细节。但无论如何，Strategy模式提供了易于理解和扩展的一种设计，至于在何种情况下使用它，就需要好好权衡一番。
<h3>Template Method(模板方法)</h3>
<p>
模板方法和Template可不一样，它相当于行为层次上的外观模式(Facade)。模板方法使得子类可以不改变一个算法的结构即可重定义该算法额度某些特定步骤，也就是说在父类就已经确定了算法的框架和原语操作，由它的子类来重载这些原语来改变具体的行为。模板方法是一种代码复用的基本技术，它们在类库中尤为重要，因为它们提取了类库中的公共行为。
</p>
<p>
下面介绍一种反向的控制结构，它是一种父类调用子类的操作，称为钩子操作(Hook Method)。它提供了缺省的行为，子类可以在必要的时候进行扩展。
</p>
{%highlight cpp%}
template <class AStrategy>
void ParentClass::Operation(){
  //ParentClass behavior
  HookOperation();
}
void ParentClass::HookOperation(){}//父类的HookOperation什么也不做
void DerivedClass::HookOperation(){
  //DerivedClass extension
}
{%endhighlight%}
使用钩子操作，就可以使得父类可以对子类的扩展方式进行控制。
<p>
还有一些需要注意的是，必须重定义的原语操作须定义为纯虚函数，模板方法自己不需被重定义，因此可以定义为一个非虚成员函数。实际上，模板方法使用继承来改变算法的一部分，而策略模式(Strategy)使用委托来改变整个算法。
</p>
<h3>Visitor(访问者)</h3>
<p>
访问者模式是整本书中的最后一种设计模式，实际上，访问者模式相当于给各个被访问的元素提供一层间接，访问者类中可以定义作用于这些元素的新操作而不改变各元素的类。访问者可以集中相关的操作而分离无关的操作，使得易于增加新的操作。
</p>

<font size="2" color="grey">要写吐了。。留一丁丁下周过来补完吧。。。</font>















