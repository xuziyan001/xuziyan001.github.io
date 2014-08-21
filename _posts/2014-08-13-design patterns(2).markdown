---
layout: post
title:  "Design Patterns(2)"
date:   2014-08-13 10:10:00
categories: Alibaba-inc update
---
<p>
马不停蹄进入结构型模式的学习，书上结构型设计模式具体分成了7类，有些类目几乎和其他设计模式一样，只是在设计的思想上与相近的设计模式有出入。由于模式种类太多，不太常用的就先一笔带过，估计最后还有一个大总结来做一个这么多设计模式的横向对比。
</p>
<p>
首先，结构型设计模式涉及到<b>如何组合类和对象以获得更大的结构</b>。结构型类模式采用继承机制来组合接口或实现，而结构型对象模式描述了如何对一些对象进行组合，从而实现新功能的一些方法。因为可以在运行时刻改变对象组合关系，所以对象组合方式具有更大的灵活性。接下来就引入7个结构型设计模式中唯一的一个类对象结构型模式--适配器模式。
</p>
<h3>Adapter(适配器)</h3>
<p>
适配器模式旨在使原本由于接口不兼容而不能再一起工作的那些类可以一起工作，这个模式在继承一些老系统的可用模块应该用得会比较多一点。实际上，适配器也可以分成<b>类适配器</b>和<b>对象适配器</b>，类适配器采用多重继承适配接口，一个分支继承接口，用另外一个分支继承接口的实现部分，在C++中，可以用<b>公共方式继承接口，用私有方式继承接口的实现</b>。
{%highlight cpp%}
class TextShape : public Shape, private TextView {
public:
  TextShape();
  virtual void BoundingBox(Point& bottomLeft, Point& topRight) const;
  virtual bool IsEmpty() const;
  virtual Manipulator* CreateManipulator() const;
};
{%endhighlight%}
一个绘图编辑器可以利用已存在的文本编辑器创建自己的文本图形，BoundBox对TextView接口进行转换使之匹配Shape的接口，而IsEmpty直接转发请求。
{%highlight cpp%}
void TextShape::BoundingBox(Point& bottomLeft, Point& topRight) const {
  Coord bottom,left,width,height;
  GetOrigin(bottom,left);
  GetExtent(width,height);
  bottomLeft = Point(bottom,left);
  topRight = Point(bottom+height,left+width);
};
bool TextShape::IsEmpty() const {
  return TextView::IsEmpty();
}
{%endhighlight%}
对象适配器采用对象组合的方法将具有不同接口的类组合在一起，在这个方法中，适配器TextShape内部维护一个指向TextView的指针。
{%highlight cpp%}
class TextShape : public Shape {
public:
  TextShape(TextView*);
  virtual void BoundingBox(Point& bottomLeft, Point& topRight) const;
  virtual bool IsEmpty() const;
  virtual Manipulator* CreateManipulator() const;
private:
  TextView* _text;
  };
{%endhighlight%}
</p>
<h3>Bridge(桥接)</h3>
<p>
桥接模式也是用得比较多的设计模式之一，特别是在Java程序中使用得尤其频繁。桥接将代码的抽象部分与它的实现部分分离，使它们可以独立的变化。抽象类的实现可以在运行时刻进行配置，将抽象与实现分离可以降低对实现部分编译时刻的依赖性，并且把实现细节隐藏起来。具体的代码就不再赘述，这里提供一个接口类如何正确得到实现类实例的方法，这里假设Window是一个接口类，它具有职责获取正确的WindowImp实例。
{%highlight cpp%}
WindowImp* Window::GetWindowImp() {
  if(_imp == 0) {
    _imp = WindowSystemFactory::Instance()->MakeWindowImp();
  }
  return _imp;
}
{%endhighlight%}
GetWindowImp负责从一个抽象工厂得到正确的实例，而抽象工厂封装了所有窗口系统的细节。实际上，桥接模式和适配器模式在某种程度上十分类似，只是前者目的是将接口部分和实现部分分离，而后者强调改变一个<b>已有</b>对象的接口。
</p>
<h3>Composite(组合)</h3>
<p>
Composite使得用户对单个对象和组合对象的使用具有一致性。用户使用Component类接口与组合结构中的对象进行交互，如果接受者是一个叶节点，则直接处理请求；如果接受者是一个Composite，它通常将请求发送给它的子部件，在转发之前或之后可能执行一些辅助操作。
{%highlight cpp%}
class Composite;
class Component {
public:
  //显式的父部件引用，需要维护一个不变式，即一个组合的所有子节点都以这个组合为父节点
  virtual Composite* GetComposite() {return 0;}//返回空指针的缺省操作
};
class Composite : public Composite {
public:
  void Add(Component*);
  void Remove(Component*)
  //...
  virtual Composite* GetComposite() {return this;}
};
class Leaf : public Component {
  //...
};
{%endhighlight%}
在设计组合模式的时候，需要注意几个问题:
<table border="0">
  <tr>1、组合模式的目的之一就是使得用户不知道他们正在使用的是Leaf还是Composite类，为了达到这一目的，我们应该最大化Component接口，并由它为这些操作提供缺省的实现。</tr><br>
  <tr>2、在没有GC的语言中，当一个Composite被销毁时，最好由Composite负责删除其子节点。</tr><br> <tr>3、Component类应该要声明一个CreateIterator操作来访问它的零件，这个操作的缺省实现返回一个NullIterator。</tr>
  
</table>
{%highlight cpp%}
Currency CompositeEquipment::Price() {
  Iterator<Equipment*> *i = CreateIterator();
  Currency total = 0;
  for(i->First();!i->IsDone();i->Next()) {
    total += i->CurrentItem()->Price();
  }
  delete i;
  return total;
}
{%endhighlight%}
在Price操作中，我们缺省遍历所有的子设备来累加设备实际价格。
</p>
<h3>Decoraor(装饰)</h3>
<p>
装饰类相当于一个Wrapper，用来动态地给一个对象添加一些额外的职责，它比静态继承要更灵活，并且避免了在层次结构高的类中拥有太多特征。
</p>
<p>
使用装饰模式需要保证装饰对象的接口必须与它所装饰的Component类的接口是一致的，所以所有的ConcreteDecorator类必须有一个公共的父类，并且这个父类要足够简单，它应集中于定义接口而不是存储数据，对数据表示的定义应延迟到子类。下面简单说明如何实现用户接口装饰：
{%highlight cpp%}
class VisualComponent {
public:
  VisualComponent();
  virtual void Draw();
  virtual void Resize();
  //...
};
class Decorator : public VisualComponent {
public:
  Decoraor(VisualComponent*);
  virtual void Draw();
  virtual void Resize();
  //...
private:
  VisualComponent* _component;
};
{%endhighlight%}
我们可以通过继承Decorator类来实现ScrollDecorator和BorderDecorator来给可是组件添加滚动和边框功能，现在我们组合这些类的实例以提供不同的装饰效果：
{%highlight cpp%}
window->SetContents(
  new BorderDecorator(
    new ScrollDecorator(textView),1
  )
);
{%endhighlight%}
怎么样，装饰模式是不是像极了适配器模式？它们的区别在于装饰仅改变了对象的职责而不改变它的接口，而适配器将给对象一个全新的接口。除此之外，装饰也相当于一个退化的、仅有一个组件的组合，但它的目的不在于对象的聚集。
</p>
<h3>Facade(外观)</h3>
<p>
外观模式有一点没看懂，书上说它是为子系统中一组接口提供一个一致的界面，使得这一子系统更加容易使用。可能是这样的吧，外观模式为一个子系统定义一个上层偏应用级别的接口，这个接口组装了这一子系统的很多部件，有时候程序的其他子系统想要访问这个子系统，但是它并不关心这个子系统是如何通过内部交互来直线功能的，所以可以提供一个单一而简单的界面，使子系统间的通信和相互依赖关系达到最小。
</p>
<p>
比如我们的IDE的编译子系统有很多组件，像Scanner接收字符流、Parser建立语法分析树、ProgramNode构造分析树、CodeGenerator产生机器代码，我们可以引入一个Compiler类将所有的部件集成在一起。
{%highlight cpp%}
class Compiler {
public:
  Compiler();
  virtual void Compile(istream&, ByteCodeStream&);
}
void Compiler::Compile(istream& input, ByteCodeStream& output) {
  Scanner scanner(input);
  ProgramNodeBuilder builder;
  Parser parser;
  parser.Parse(scanner,builder);
  RISCCodeGenerator generator(output);
  ProgramNode* parseTree = builder.GetRootNode();
  parseTree->Traverse(generator);
}
{%endhighlight%}
当然，编译器的外观还可以对Scanner和ProgramNodeBuilder这样的参与者进行参数化以正价系统的灵活性，但是这并非外观模式的主要任务，它的任务是为一般情况简化接口。并且通常来讲，仅需要一个Facade对象，因此Facade对象属于Singleton模式。
</p>
<h3>Flyweight(享元)</h3>
<p>
Flyweight的意思是“蝇量级”，由名字可以看出这一设计模式旨在减少大程序的开销。
</p>
<p>
享元模式运用共享技术有效地支持大量细粒度的对象，这里，flyweight是一个共享对象，它可以在多个场景下使用(context)，并且在每个场景中都可以作为一个独立的对象，它的关键概念是<b>内部状态</b>和<b>外部状态</b>之间的区别。内部状态存储于flyweight中，它包含了独立于flyweight场景的信息，它可以被共享；而外部状态取决于Flyweight场景，并根据场景而变化，因此不可共享。用户对象负责在必要的时候将外部状态传递给flyweight。
</p>
<p>
无奈此模式的代码太多，这里就不贴出来了。享元模式在文档编辑器中使用得比较多，它不用为了每一个字符创建一个单独的对象，只需在FlyweightFactory中保持一个相关的关联存储，仅在每次被调用的时候实例化一个新对象，这样就可以大幅度减少程序的存储开销。享元模式通常和组合模式结合起来，用共享叶节点的有向五环图实现一个逻辑上的层次结构。
</p>
<h3>Proxy(代理)</h3>
<p>
代理模式比较好理解，就是为其他对象提供一种代理以控制对这个对象的访问，为两个对象之间提供了一定程度的间接性，这一层间接性可以用来做很多事情，比如说远程代理(Remote Proxy：为一个对象在不同地址空间提供局部代表)、虚代理(Virtual Proxy：将开销很大的对象延迟到需要时创建)、保护代理(Protection Proxy：控制对原始对象的访问，提供一层授权)或智能指引(Smart Reference：著名的智能指针啊，C++11的智能指针就使用了代理模式提供对实际对象的引用计数，但它包括但不限于在智能指针上的应用)。
</p>
<p>
代理模式还可以对用户隐藏另一种称之为<b>copy-on-write</b>的优化方式。拷贝一个庞大而复杂的对象是一种开销很大的操作，如果这个拷贝根本就没有被修改，那么这些开销就没有必要，使用代理方式延迟这一拷贝过程可以大大的减少程序的运行开销(虚代理)。不止如此，实现copy-on-write时还需对实体进行引用计数。拷贝代理增加引用计数，当用户请求一个修改该实体的操作时，代理才会真正拷贝它，这种情况下，代理还得负责减少实体的引用计数(智能指引)。
下面定义一个Image的代理ImageProxy来延迟Image实体的创建。
{%highlight cpp%}
class ImageProxy : public Graphic {//Graphic也是Image的父类，ImageProxy接口与Image一致
public:
  ImageProxy(const char* imageFile);
  virtual Point& GetExtent();
  //...
protected:
  Image* GetImage();
private:
  Image* _image;
  Point _extent;
  char* _fileName;
}
ImageProxy::ImageProxy(const char* fileName) {
  _fileName = strdup(fileName);
  _extent = Point::Zero;
  _image = 0;
}
Image* ImageProxy::GetImage() {//提供晚实例化
  if(_image == 0) {
    _image = new Image(_fileName);
  }
  return _image;
}
const Point& ImageProxy::GetExtent() {
  if(_extent == Point::Zero) {
    _extent = GetImage()->GetExtent();
  }
  return _extent;
}
{%endhighlight%}
</p>
<h3>最后</h3>
<p>
好啦，结构型设计模式又告一段落了，看结构型设计模式的时候明显比之前看创建型模式要快很多，总共花了两天上班的间歇期看书，一天的时间总结发文，这个效率还是很不错的！(好吧，说是间歇期，今天已经花了差不多3个小时在这个上面)
</p>
<p>
接下来就要进入行为模式这一章了，这一章设计模式比较多，估计其中各设计模式的相似度更高，所以估计理解起来不是那么容易，预计一周的时间把它看完，然后结束对所有设计模式的介绍吧，之后还要给博客添加一个评论功能，虽然应该不会有人评论吧，但是万一就火了呢。。。嗯。。。
</p>

<font size="2" color="grey">洋洋洒洒好几千字，累死我了。。。</font>












