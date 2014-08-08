---
layout: post
title:  "Design Patterns(1)"
date:   2014-08-06 10:31:00
categories: Alibaba-inc update
---
<p>
可复用面向对象的设计模式大概可以分成三类，即<b>创建型模式</b>、<b>结构型模式</b>和<b>行为模式</b>。今天来复述一下创建型模式中的5种方法。其实看了前三种方法之后，并没有找到它们之间的本质区别，这些玩意啃起来也实在是太吃力了，只好一边写，一边回顾这些模式到底都干了些啥。
</p>
<h3>Abstract Factory(抽象工厂)</h3>
<p>
抽象工厂的意图是提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。<br>
什么意思呢？应该是通过AbstractFactory仅声明一个创建产品的接口，产品的每个组件都有一个抽象类，而具体的子类则用来实现产品特定的风格。利用书上创建迷宫的例子来说：
{% highlight cpp %}
class MazeFactory{
public:
  MazeFactory();
  virtual Maze* MakeMaze() const
    {return new Maze;}
  virtual Wall* MakeWall() const
    {return new Wall;}
  virtual Room* MakeRoom(int n) const
    {return new Room(n);}
  virtual Door* MakeDoor(Room* r1, Room* r2) const
    {return new Door(r1,r2);}
};
{% endhighlight %}
类MazeFactory作为一个抽象工厂，它建造房间，墙壁，房间之间的门，建造迷宫的程序将MazeFactory作为一个参数，就可以指定要创建迷宫的所有组件。当我们要创建另一个魔法型迷宫时，我们可以创建实例工厂EnchantedMazeFactory去重定义不同的成员函数并返回组件的不同子类。
{% highlight cpp %}
class EnchantedMazeFactory : public MazeFactory{
public:
  EnchantedMazeFactory();
  virtual Room* MakeRoom(int n) const
    {return new EnchantedRoom(n,CastSpell());}
  virtual Door* MakeDoor(Room* r1, Room* r2) const
    {return new DoorNeedingSpell(r1,r2);}
protected:
  Spell* CastSpell() const;
};
{% endhighlight %}
这样，CreateMaze就可以接受一个EnchantedMazeFactory实例来建造魔法型迷宫：
{% highlight cpp %}
MazeGame game;
EnchantedMazeFactory factory;
game.CreateMaze(factory);
{% endhighlight %}
</p>
<h3>Builder(生成器)</h3>
<p>
生成器将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。这句话读起来真是拗口极了，简单说来，生成器就是分离你的构造代码和表示代码，使你可以对构造过程进行更精细的控制，并且可以改变一个产品的内部展示。<br>
通常，一个抽象的Builder类为导向者可能要求创建的每一个构件定义一个操作，这些操作缺省什么都不做，由ConcreteBuilder类对它有兴趣创建的构件重定义这些操作，现在我们以Builder模式创建一个迷宫：
{% highlight cpp %}
class MazeBuilder{
public:
  virtual void BuildMaze(){}
  virtual void BuildRoom(int room){}
  virtual void BuildDoor(int roomFrom, int roomTo){}
  virtual Maze* GetMaze(){return 0}
protected:
  MazeBuilder(); 
  //受保护的构造函数，不能自己创建，只通能过子类进行创建？
};
{% endhighlight %}
子类StandardMazeBuilder是一个创建迷宫的简单实现：
{% highlight cpp %}
class StandardMazeBuilder : MazeBuilder{
public:
  StandardMazeBuilder();
  virtual void BuildMaze();
  virtual void BuildRoom(int room);
  virtual void BuildDoor(int roomFrom, int roomTo);
  virtual Maze* GetMaze();
private:
  Direction CommonWall(Room*, Room*);
  Maze* _currentMaze;
};
{% endhighlight %}
现在，我们就可以用CreateMaze和StandardMazeBuilder来创建一个迷宫：
{% highlight cpp %}
Maze* maze;
MazeGame game;
StandardMazeBuilder builder;
game.CreateMaze(builder);
maze = builder.GetMaze();
{% endhighlight %}
若需要创建另一个迷宫，就可以在MazeGame类中定义一个CreateAnotherMaze函数：
{% highlight cpp %}
Maze* MazeGame::CreateAnotherMaze(MazeBuilder& builder)
{% endhighlight %}
用来分离迷宫的表示和实际构造情况。
</p>
<h3>Factory Method(工厂方法)</h3>
<p>
其实上述抽象工厂就是使用工厂方法实现的，工厂方法定义一个用于创建对象的接口，让子类来决定实例化哪一个类。这里介绍一个我个人觉得很酷的，使用模板来避免创建子类的方法。<br>
工厂方法有一个潜在的问题就是可能仅为了创建合适的Product对象而迫使你创建Creator子类，这时，我们可以提供一个Creator的模板子类，它使用Product类作为模板参数：
{% highlight cpp %}
class Creator{
  public:
    virtual Product* CreateProduct() = 0;
};
template<class TheProduct>
class StandardCreator : public Creator{
public:
  virtual Product* CreateProduct();
};
template<class TheProduct>
Product* StandardCreator<TheProduct>::CreateProduct(){
  return new TheProduct;
}
{% endhighlight %}
使用这个模板，就可以仅提供产品类，而不需要创建Creator的子类。
{% highlight cpp %}
class MyProduct : public Product{
public:
  MyProduct();
  //...
};
StandardCreator<MyProduct> myCreator;
{% endhighlight %}
</p>
<h3>Prototype(原型)</h3>
<p>
使用一个原型实例指定创建对象的种类，然后通过拷贝这些原型创建新的对象，原型的子类都需要实现一个Clone操作。原型模式最大的困难在于如何去正确实现Clone操作，使用深拷贝还是浅拷贝？对象结构包含循环引用如何处理？Clone迫使你决定如果所有东西都被共享了该怎么办。有时候，还需要为原型构件新增一个Initialize操作对一些内部值进行初始化工作。<br>
现在我们定义一个MazePrototypeFactory，它使用要创建的对象的原型来初始化所有构件。
{% highlight cpp %}
class MazePrototypeFactory : public MazeFactory{
public:
  MazePrototypeFactory(Maze*, Wall*, Room*, Door*);
  virtual Maze* MakeMaze() const;
  virtual Room* MakeRoom() const;
  virtual Wall* MakeWall() const;
  virtual Door* MakeDoor() const;
private:
  Maze* _prototypeMaze;
  Room* _prototypeRoom;
  Wall* _prototypeWall;
  Door* _prototypeDoor;
};
{% endhighlight %}
用于创建墙壁、房门等的成员函数式相似的，每个成员函数都要克隆一个原型，然后进行初始化。
{% highlight cpp %}
Door* MazePrototypeFactory::MakeDoor(Room* r1, Room* r2) const {
  Door* door = _prototypeDoor->Clone();
  door->Initialize(r1, r2);
  return door;
}
//...
{% endhighlight %}
现在只需要使用基本迷宫构件的原型进行初始化，就可以由MazePrototypeFactory来创建一个不同原型集合的迷宫类型。
{% highlight cpp %}
MazeGame game;
MazePrototypeFactory enchantedMaze(
  new Maze, new enchantedWall, new enchantedRoom, new enchantedDoor
  );
Maze* maze = game.CreateMaze(EnchantedMaze);
{% endhighlight %}
值得注意的是，每个构件子类的Clone函数声明都是返回原型类型的指针，实现中自动进行向下转型，避免了调用Clone的时候还需要显式的指出所需转换的类型，并且还必须有一个拷贝构造器用于克隆。
{% highlight cpp %}
Wall* enchantedWall::Clone() const {
  return new enchantedWall(*this);
}
enchantedWall::enchantedWall (const enchantedWall& other) : Wall(other) {
  _enchantedWall = other._enchantedWall;
}
{% endhighlight %}
当一个系统中的原型数目不固定时，最好是可以保持一个可用原型的注册表来对原型进行更好的管理。
</p>
<h3>Singleton(单例)</h3>
<p>
单例模式应该是听说得最多的设计模式之一了吧，但是一直不懂究竟什么样的设计才能被称为单例模式设计，书上给出的定义是“保证一个类仅有一个实例，并提供一个访问它的全局访问点”、“让类自身负责保存它的唯一实例，这个类可以保证没有其他实例可以被创建，并且它可以提供一个访问该实例的方法”。需要注意的是：
<table border="0">
  <tr>1、为了保证静态对象只有一个实例会被声明，可以使用静态成员函数返回唯一实例。</tr><br>
  <tr>2、使用一个单例注册表(registry of singleton)，Singleton类可以根据名字在一个众所周知注册表中注册它们。</tr>
</table>
{% highlight cpp %}
class Singleton{
public:
  static void Register(const char* name, Singleton*);
  static Singleton* Instance();
protected:
  static Singleton* Lookup(const char* name);
private:
  static Singleton* _instance;
  static List<NameSingletonPair>* _registry;
};
{% endhighlight %}
假定一个环境变量指定了所需要单例的名字，那么Instance()函数就可以根据获取环境变量来进行查找。
{% highlight cpp %}
Singleton* Singleton::Instance(){
  if(_instance == 0){ //惰性初始化(Lazy Initialize)
    const char* singletonName = getenv("SINGLETON");
	_instance = Lookup(singletonName);
	//没有此单例则返回0
  }
  return _instance;
}
{% endhighlight %}
其他单例可以在自己的构造函数中注册自己，但除非实例化类否则它们的构造函数不会被调用。在C++中，我们可以在包含MySingleton实现的文件中定义：
{% highlight cpp %}
static MySingleton mySingleton
{% endhighlight %}
来避免这个问题。
</p>
<h3>最后</h3>
<p>
因为是一边翻着书，一边把觉得有用的东西往上贴的，不免看起来乱乱的，还有这个奇怪的代码高亮以及字体设置排版等问题，都需要去解决一下，这些都在之后慢慢改进吧，还有是不是应该弄一些比较炫酷的功能呢？<br>
这个网站实际上是以陈皓师兄的coolshell为目标的，结果上阿里内外一搜，发现陈皓的主管竟然是章文嵩，当时章文嵩跟我们培训的时候我还觉得他萌萌的呢。。。
</p>
<font size="2" color="grey">Whatever, 今天周五了，啊哈哈哈哈。。。</font>







