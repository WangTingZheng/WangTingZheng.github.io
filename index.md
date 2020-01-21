# Room改进

上一讲我们已经利用`Entity`、`Dao`、`Database`三个类定义了数据库，并且在`MainActivity`中调用数据库的增删改查操作，根据按键事件来进行相应的操作，并且协调`TextView`显示结果，但是这种原始的写法有很多弊端：

- 每次修改数据库的值之后都要执行一次更新操作，很麻烦

- 数据库的创建很浪费时间，多次新建数据库对象耗费资源太多

-  我们只能让创建数据库的操作在主线程中执行，这十分耗费系统资源，容易造成页面卡顿
- 所有的逻辑代码都写在`MainActivity`中，它承担了它不应该承担的功能

针对以上的缺陷，我们将在本节中尝试改进，大概的改进方法如下：

- 利用LiveData来存储数据库返回的数据，并且通过观察者模式来对数据库中数据库的变更采取动作

- 数据库的创建和调用采用单例模式，只让`Database`只能产生一个能被公共访问的数据库对象
- 把数据库的增删改查操作改为异步模式
- 引入`ViewModel`来管理界面相关的工作，引入`Repository`来管理数据相关的工作

## 采用LiveData代替List

### observer模式介绍

observer pattern，中文名观察者模式，它是一种设计模式，它的定义是：

> 指多个对象间存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

观察者模式的提出，就是Push代替Poll的结果，所谓Poll，就是轮询，观察者按照一定的时间频率询问被观察者是否有数据更新，如果有，就接收数据来更新自己。这种机制有很大缺陷，第一个问题是：观察者观察被观察者的时间频率设置在多少是合适的？1秒钟？如果被观察者在1秒钟内有了多次更新了呢？第二个问题：如果观察者的数量很大呢？那么被观察者将承受很多次询问，观察者通常还不止一个，毫无疑问，数量巨大的观察者对被观察者采取高速的查询操作，对被观察者造成的压力是很大的。

所以就有了Push模式来代替Poll：

![Observer](./Observer.png)

我们可以看到，一个被观察者对应多个观察者，被观察者中有一个叫`Observers`的列表，它保存这与它相关的所有观察者，以便数据有改变后逐个通知，列表的观察者可以使用`add(IObserver)`和`remove(IObserver)`来增加、减少。当被观察者的状态发生改变时，它执行`notify()`来逐个通知在`Observers`列表内的所有观察者；在一个观察者中，只有一个`updata()`来执行当被观察者状态发生改变时自身的更新，令人惊奇的是，这个函数居然是无参数的，也就是说它不知道被观察者改变的数据是什么，那它是根据什么采取相应的动作来更新呢？这个问题我们将在下面的讲解中揭晓。

我们可以看到，有一个箭头从被观察者指向观察者，这代表一个被观察者对应多个观察者，这一点也能体现在`Observers`列表里，那么下面对象之间的箭头是怎么回事？我们先不着急，先实现两个接口。

对于被观察者，我们需要实现的是`add()`，`remove()`，`notify()`，前面两个就不多说，这与你储存`Observers`列表的数据结构有关。对于`notify()`，我们可以如下实现：

```java
public interface IObservale{
    List IObservers;
    void add(IObserver o);
    void remove(IObserver o);
    void notifty();
}

public interface IObserver{
    void update();	
}

public class ConcreteObservable implements IObservable{
    public int data;
    List concreateObserverList; //用来储存观察被观察者的观察者
    void add(ConcreteObserver concreteObserver){
        ...
    }
    void remove(ConcreteObserver concreteObserver){
        ...
    }
    void notify(){  //从列表中那到所有观察者，逐个执行它们的更新函数
        for(int a = 0; a< concreateObserverList.size(); a++){
            concreateObserverList.get(a).update()
        }
    }
    public int getDate(){  //获得被观察者的数据/状态
        return this.data;
    }
}

public class ConcreteObserver implements IObserver{
    ConcreteObservable concreteObservable;  //被观察者对象，用来传递数据信息
    public concreteObserver(ConcreteObservable concreteObserable){  //构造方法
        this.concreteObservable = concreteObservable;//传入观察者所观察的被观察者到观察者对象
    }
    public update(){  //更新函数，根据数据情况采取相应的措施
        int data = this.concreteObservable.getData(); //通过传入的被观察者的getData函数获取数据
        ...
    }
}
```

我们可以看到，观察者模式巧妙地利用传入观察者的被观察者，传入了数据，这样就不用再update内传入参数了。我们同时回答了上文的两个问题：为什么两个类之间有箭头和为什么update函数没有参数。

我们在使用观察者模式的时候，我们可以通过下面的代码来创建被观察者、观察者，连接两者：

```java
ConcreteObservable Observable = new ConcreteObserable()
ConcreteObserver Observer1 = new ConcreteObserver(Observable)
ConcreteObserver Observer2 = new ConcreteObserver(Observable)
Observable.add(Observer1)
Observable.add(Observer2)
```

### 数据库数据储存改造

当然，你也可以直接给`updata()`传递参数，LiveData里的观察者模式就是这种。我们先把Dao里的获取数据库所有数据的函数改为LiveData储存：

```java
@Query("SELECT * FROM WORD ORDER BY ID DESC")
LiveData<List<Word>> getAllWordLive();
```

这样，返回的LiveData就能作为一个被观察者了，现在我们要在`MainActivity`里创建一个观察者，来观察这个返回的LiveData数据，如果它有更新，观察者将采取必要的动作：

```java
wordViewModel.getAllWordLive().observe(this, new Observer<List<Word>>() {
    @Override
    public void onChanged(List<Word> words) {
        String text = "";  //define a value to store data from database
        for (int i = 0; i < words.size(); i++) {   //fetch all data from database
            Word word = words.get(i);
            text += word.getId() + ":" + word.getWord() + "=" + word.getChineseMeaning() + "\n";
        }
        textView.setText(text);   //show database in textView
 }
```

这里的`wordViewModel.getAllWordLive()`就是返回我们要观察的被观察者，之后的`.observer()`相当于前面的`add()`，所以里面填写的自然是新创建的观察者：`new Observer<List<Word>()`。接下来，我们要写观察者的`update()`函数，我们要对变化的数据进行处理，这里的处理就是直接把它显示在`textView`中，这样当数据库的数据发生改变时，LiveData就会发生改变，然后被观察者就会执行观察者的`update()`函数，当然这里是`onChanged()`函数，这样新的数据就会被显示在`textView`上。

## 改进数据库操作

上文我们提到，在app主线程上创建数据库进行增删改查操作时，可能会导致程序主线程卡顿，而这种卡顿很容易被用户察觉到，从而影响用户体验，在本节中，我们将尝试把数据库的增删改查操作放在异步线程中去执行，这样这些操作就会被安排在后台的异步线程里运行。

```java
static class UpdateAsyncTask extends AsyncTask<Word, Void, Void> {
    private WordDao wordDao;

    UpdateAsyncTask(WordDao wordDao) {
        this.wordDao = wordDao;
    }

    @Override
    protected Void doInBackground(Word... words) {
        wordDao.updateWords(words);
        return null;
    }
}
```

构造方法负责把要处理的数据传入线程，`doBackgroud()`函数负责在后台异步线程上执行数据处理操作，当我们执行

```Java
new UpdateAsyncTask(wordDao)
```

时，我们将创建一个数据库更新类，当我们运行这个类的`execute()`方法时，系统会主动在后台线程上运行事先设计好的`doInBackground()`函数。

之后，我们可以给一个函数来规范地调用数据库更新操作

```java
void updateWords(Word... words) {
    new UpdateAsyncTask(wordDao).execute(words);
}
```

## 采用singleton来改进Database类

### singleton介绍

singleton pattern，中文名单例模式，是一种设计模式，它的定义是：

> 单例模式的类只能实例化一个能被全局访问的对象

这里有两个要点：

- 只能实例化一个对象
- 该对象能被全局访问

我们知道，类的实例化通常与它的构造方法有关，比如说有下面一个类：

```java
public class SingletonDemo{
    public SingletonDemo(){
        
    }
}
```

理论上，我们只要执行：

```java
SingletonDemo singletonDemo = new SingletonDemo();
```

就能实例化出该类的一个对象，这是通常的方法，明显的，我们可以创建无数个对象，那么如何才能只创建一个对象呢？我们知道到，我们上面采用的实例化方法通常是在类的外部实例化这个类，到目前为止，这种实例化我们是无法控制的，所以在singleton的设计中，我们需要禁止这种行为，做法很简单，把这个类的构造方法改为私有的：

```java
public class SingletonDemo{
    private SingletonDemo(){
        
    }
}
```

这样你就不能通过new的方法实例化这个类了，这个类只能由它自己实例化。那么问题来了：你如何实例化一个只能由自己实例化自己的类呢？这里就要使用到静态方法了：

```java
public class SingletonDemo{
    private static SingletonDemo singletonDemo;
    private SingletonDemo(){
        
    }
    static SingletonDemo getSingletonDemo(){
        if(singeltonDemo == null){
        	singeltonDemo = new SingletonDemo();
        }
        return singeletonDemo;
    }
}
```

我们可以看到，这个类的构造方法是`private`的，所以它不能由外部类来实例化，只能由它自己实例化，而在静态方法`getSingletonDemo()`中，我们将检测对象是否存在，如果不存在，则实例化一个对象，如果存在，这返回旧的对象，由于这个方法是static的，属于这个类的作用域，所以它可以访问到这个类中的static变量，也就能直接实例化这个类了，而且这个对象是唯一的，因为第一次实例化以后，函数将返回旧的对象，不再实例化类。

接下来我们应该让这个对象能在全局下被访问，我们只需要让`getSingletonDemo()`能被全局访问就行了，代码修改如下：

```java
public class SingletonDemo{
    private static SingletonDemo singletonDemo;
    private SingletonDemo(){
        
    }
    public static SingletonDemo getSingletonDemo(){
        if(singeltonDemo == null){
        	singeltonDemo = new SingletonDemo();
        }
        return singeletonDemo;
    }
}
```

这样我们就能得到一个符合单例模式的类了。在本例中，我们还得考虑多线程的问题，在多线程的情况下，还是有可能出现实例化多个对象的情况，所以我们要给`getSingletonDemo`加一个锁：

```java
public class SingletonDemo{
    private static SingletonDemo singletonDemo;
    private SingletonDemo(){
        
    }
    public synchronized static SingletonDemo getSingletonDemo(){
        if(singeltonDemo == null){
        	singeltonDemo = new SingletonDemo();
        }
        return singeletonDemo;
    }
}
```

当然，这不是单例模式的全部，此种单例模式还有很多缺陷，你如果想了解更多更详细的单例模式，可以查阅[这篇文章](https://www.cnblogs.com/cielosun/p/6582333.html)。

### 改进数据库创建

结果上面的数据库异步处理，我们就能正常创建数据库了，加上上面介绍的单例模式，我们就能把`WordDao.class`数据库的创建操作改装成可在后台运行的单例模式了：

```java
private static WordDatabase INSTANCE;

static synchronized WordDatabase getDatabase(Context context) {
    if (INSTANCE == null) {
        INSTANCE = Room.databaseBuilder(context.getApplicationContext(), WordDatabase.class, "word_database")
            .build();
    }
    return INSTANCE;
}
```

## 优化任务分配

在目前为止，所有的数据库创建、数据库操作、响应页面的操作都是由`MainActivity`这个类来完成的，这显得很难维护，在本节中，我们将尝试把数据库的数据操作相关的任务分给`Repository`来管理，把UI的数据响应交给`ViewModel`来管理，`MainActivity`只完成UI的响应中的协调的工作。

我们只需要把四个数据库的增删改查操作类和具体的它的方法移到一个叫`WordRepository`的类中，这个类的构造方法能产生数据库增删改查所需要的Dao类，当这个类被新建时，传入的参数是Activity，通过这个可以根据`WordDatabase.class`创建Database，通过这个数据库，我们可以Dao，这样就可以进行增删改查的操作了。

在`WordViewModel`中，我们可以继续使用上面的三个增删改查的函数，因为现在UI涉及的操作也只有数据库的增删改查的操作。

## MainActivity的改写

现在它的功能就只剩下协调UI和viewModel来完成UI的响应了，所以我们只需要在里面新建各个UI的变量，再与页面绑定UI组件，再为各个按键编写`onClick()`函数，这个函数自然就是调用viewModel的函数处理、更新数据到另外的UI也就是textView中了。

```java
buttonInsert.setOnClickListener(new View.OnClickListener() {  
    @Override
    public void onClick(View v) {
        Word word1 = new Word("Hello", "你好");  //new a database data
        Word word2 = new Word("World", "世界");
        wordViewModel.insertWords(word1, word2);
    }
});
```

## 总结

这样优化过的app各个部分分工明确，好维护而且通过引入LiveData、singleton、数据库增删查改异步化，大大增强了程序运行的稳定性和效率，我们可以把上面的改进成果归纳2成下面的这幅图：

![总结](./room设计模式.png)

