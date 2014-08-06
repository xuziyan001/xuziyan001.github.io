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
{% highlight ruby %}
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
{% highlight ruby %}
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
{% highlight ruby %}
MazeGame game;
EnchantedMazeFactory factory;
game.CreateMaze(factory);
{% endhighlight %}
</p>
<h3>Builder(生成器)</h3>
<p>
生成器将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。这句话读起来真是拗口极了，简单说来，生成器就是分离你的构造代码和表示代码，使你可以对构造过程进行更精细的控制，并且可以改变一个产品的内部展示。<br>
通常，一个抽象的Builder类为导向者可能要求创建的每一个构件定义一个操作，这些操作缺省什么都不做，由ConcreteBuilder类对它有兴趣创建的构件重定义这些操作，现在我们以Builder模式创建一个迷宫：
{% highlight ruby %}
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
{% highlight ruby %}
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
{% highlight ruby %}
Maze* maze;
MazeGame game;
StandardMazeBuilder builder;
game.CreateMaze(builder);
maze = builder.GetMaze();
{% endhighlight %}
若需要创建另一个迷宫，就可以在MazeGame类中定义一个CreateAnotherMaze函数：
{% highlight ruby %}
Maze* MazeGame::CreateAnotherMaze(MazeBuilder& builder)
{% endhighlight %}
用来分离迷宫的表示和实际构造情况。
</p>
<h3>Factory Method(工厂方法)</h3>
<p>
其实上述抽象工厂就是使用工厂方法实现的，工厂方法定义一个用于创建对象的接口，让子类来决定实例化哪一个类。这里介绍一个我个人觉得很酷的，使用模板来避免创建子类的方法。<br>
工厂方法有一个潜在的问题就是可能仅为了创建合适的Product对象而迫使你创建Create子类，这时，我们可以提供一个Creator的模板子类，它使用Product类作为模板参数：
{% highlight ruby %}
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
{% highlight ruby %}
class MyProduct : public Product{
public:
  MyProduct();
  //...
};
StandardCreator<MyProduct> myCreator;
{% endhighlight %}
</p>
<font size="2" color="grey">未完。。。</font>