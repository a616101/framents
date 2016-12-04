翻译自 [Flaw: Constructor does Real Work](http://misko.hevery.com/code-reviewers-guide/flaw-constructor-does-real-work/)

<br>
我们往往在构造器（constructor）中做以下工作：
* 创建和初始化 collaborators??
* 与其他service通信
* set up its own state removes seams needed for testing??
* 以上促使强制子类或者模仿对象（mocks）继承了不想继承的行为（behavior）

在构造器中做了太多事情会阻止了实例化，或者在测试中更换collaborators


### 出现以下情况就是警告了

* 在构造器或者在字段声明中出现new关键字
* 在构造器或者在字段声明中调用静态方法
* 在构造器中有除了字段定义其他的事情
* 构造器结束的时候对象没有初始化完全（注意初始化方法）
* 在构造器中有控制流程（条件语句或者循环逻辑）
* 在构造器中使用控制流程做了很复杂的创建对象，而不是使用工厂方法（factory）和生成器
* 添加或使用一个初始化块

### 为什么这是一个缺陷？
当你的构造器不得不实例化和初始化它的collaborators时，会导致一个不灵活的和过早的 耦合设计。这样的构造器切断了在测试时注入测试collaborators的能力。

#### 它违反了单一职责原则
当collaborators构造器与初始化混在一起，它暗示着，只有一个方法去配置这个类，这封闭了重用的机会。对象图创建是一个完备的职责 - 不同于为什么一个类出现在第一位。在构造器中做这样的工作违反了单一职责原则。

#### 直接测试很困难
测试这样的构造器是很困难的。要实例化一个对象，构造器必须要执行。然后如果这个构造器做了很多工作，你不得不在测试中创建这个对象时做很多工作。如果collaborators接入了外部资源（比如文件，网络服务，数据库），collaborators中微小的改变都会反映在构造器中，然而可能会被漏测因为测试没有被覆盖到，而测试没有被覆盖到是因为构造器太难测所以用例没有写到某些点。最后我们陷入了这样的恶性循环。

#### 使用子类和覆写做测试依旧是有缺陷的
某些情况，一个构造器做少量的工作，然后委托于一个期待被测试子类覆写的方法去做。这可以解决这个复杂构造器的问题，但是通过测试子类的做法应该是你最后才用到的招数。另外，通过子类的方式，你没办法测试被你覆写的方法。而且，这个方法做了很多工作（记住这就是为什么它要在第一时间创建），所以它应该被测试。

#### 它迫使Collaborators on You
有时，当你测试一个对象，你并不想要创建它所有的collaborators。举个例子，当你要连接mySql服务时，你不想创建一个真正的mySqlRepository 对象。然后，在测试中的系统（System Under Test SUT)里你通过new MySqlRepositoryServiceThatTalksToOtherServers()创建，那么你会拥有一个很重的对象。

#### 它去掉了“Seam”
Seam是一个你能将你的代码库分离成依赖和初始化一个小而聚集的对象。当你在构造器中使用new XYZ(),你永远不会创建一个不同的（子类）对象。（阅读这本书获取更多关于Seam的知识[Working Effectively with Legacy Code](https://www.amazon.com/Working-Effectively-Legacy-Robert-Martin/dp/0131177052))。就算你使用多样化构造器（有些只为了测试）去创建一个单独的用于测试的构造器，也不能解决问题。这个在工作的构造器依然将要被其他类使用。就算你能在这个隔绝环境（通过创建测试专属的构造器）测试这个对象，你还是会运行那些很难测试的构造器的类。然后当你测试那些很难测试的类，你将无能为力。

#### 底线
最后都落在了这个问题上：在隔离环境或者通过test-double collaborators 构建一个类有多容易或者多难。
* 如果很难，你在构造器中做了太多工作。
* 如果简单，相信自己。

在写代码的时候要一直考虑测这个对象会有多难。通过你写的这个构造器实例化会很容易么？（注意你的
测试类不是这个类唯一会被实例化的地方）

> 有很多的设计都是“实例化其他对象或者在全局路径中获得对象 的对象。这个编程实践，如果没有被检查到，将导致难以测试的高耦合设计。[J.B. Rainsberger, JUnit Recipes, Recipe 2.11]


<br>
### 认出缺陷的地方

#### 检查以下症状

* 用new关键字构建任何你想在测试中替换的test-double？（通常这不仅仅是比一个简单的值对象更大）
* 调用了静态方法？（记住，静态调用是不可模仿的，不可注入的，所以如果你看见`Server.init()`类似的，警报应该在你脑中响起）
* 有条件或者循环逻辑？（你将在每次实例化对象的时候都会执行这段逻辑。这会导致无论是在你直接测试这个类，还是在你测试相关类时正好需要这个类，都会有过重的设置代码）。

#### 当你写或者检查代码的时候思考考虑这样一个基础的问题
我将要怎么测试它？

> 如果答案不明确，或者好像测试将看起来很丑，或者测试将很难去写，那么这就是一个警告信号。你的设计可能需要修改；改变一些东西直到到代码很容易测试，这时你付出的努力将使这段代码比之前远远好很多。[Hunt, Thomas. Pragmatic Unit Testing in Java with JUnit]

构建值对象（value objects）在很多情况下都可以接受（比如，链表，哈希图，用户，邮箱地址，信用卡），值对象的主要特点是：（1）简单的构建（2）关注于状态(大量少行为的getter和setter方法）（3）不指向任何服务（service）对象
<br>

### 解决这些缺陷
不要在构造器中创建collaborators，而是把它们传入进去

将对象图构造器的职责移除，在另一个对象中初始化它。（取出作为一个生成器、工厂或者提供商，然后将这些collaborators传给你的构造器）。

比如：如果你依赖一个DatabaseService（还好它是一个接口），然后用依赖注入，将你所需要的那个DatabaseService子类的对象传入到构造器中。

重申：不要在你的构造器中创建collaborators，而是将它们传入进去。
如果你需要在初始化的时候传入对象，你有以下三个选择：
1. 使用Guice的最好的方式：使用Provider<YourObject>去创建和初始化YourObject的构造器变量。将对象初始化和图构造器的职责留给Guice。这将避免在进行中时候初始化这个对象。又是你将需要用到生成器或者工厂，而不是提供商，然后将该生成器和工厂传给构造器。
2. 使用手动依赖注入的最好的方式：使用一个生成器，或者一个工厂，作为你的YourObject的构造器参数。通常情况下，有一个工厂生成整个对象图，看一下的例子。（所以你不需要担心因为给每个类一个工厂所以会有类爆炸的情况）这个工厂的职责是创建一个对象图，然后什么也不做。（这个工厂中的全部就是整个的一堆new关键字和传送引用)。这个对象图的职责就是完成任务，而不做对象实例化（在应用逻辑类中，应当只有很少的new关键字）。
3. 仅仅作为最后一个手段：在你的类中有一个`init(…)`方法，然后在构造之后调用。尽量避免这种方式，最好使用另一个单一职责的对象来为这个对象配置参数。（如果你在用Guice，这可能是一个提供商）

（请阅读以下的代码）
<br>

### 具体的代码例子 之前和之后
基本上，“在构造器中工作”使实例化你的对象很困难，或者使引入test-double对象很困难。

---------
问题：在构造器中使用new关键字来定义字段

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Basic new operators called directly in
//   the class' constructor. (Forever
//   preventing a seam to create different
//   kitchen and bedroom collaborators).
class House {
  Kitchen kitchen = new Kitchen();
  Bedroom bedroom;

  House() {
    bedroom = new Bedroom();
  } 

  // ...
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>class House {
  Kitchen kitchen;
  Bedroom bedroom;

  // Have Guice create the objects
  //   and pass them in
  @Inject
  House(Kitchen k, Bedroom b) {
    kitchen = k;
    bedroom = b;
  }
  // ...
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// An attempted test that becomes pretty hard

class HouseTest extends TestCase {
  public void testThisIsReallyHard() {
    House house = new House();
    // Darn! I'm stuck with those Kitchen and
    //   Bedroom objects created in the
    //   constructor. 

    // ...
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// New and Improved is trivially testable, with any
//   test-double objects as collaborators.

class HouseTest extends TestCase {
  public void testThisIsEasyAndFlexible() {
    Kitchen dummyKitchen = new DummyKitchen();
    Bedroom dummyBedroom = new DummyBedroom();

   &nbsp;House house =
        new House(dummyKitchen, dummyBedroom);

    // Awesome, I can use test doubles that
    //   are lighter weight.

    // ...
  }
}</pre>
</td>
</tr>
</tbody>
</table>

这个例子混合了对象图创建和逻辑。测试中我们想创建一个与在生产环境上不同的对象图。通常是一个有一些被test-double替换的对象的图。new操作内联时我们将永远不可能为测试创建一个对象图。参考[“How to think about the new operator”](http://misko.hevery.com/2008/07/08/how-to-think-about-the-new-operator/)

* 缺陷：通过内联对象实例化定义字段，与在构造器实例化对像有同样的问题。
* 缺陷：这会使初始化很容易，但如果Kitchen是一些比较“昂贵”的东西，比如文件或者数据库读取，由于我们不能将 Kitchen 或者 Bedroom 替换成一个test-double，所以将不容易测试。
* 缺陷：你的设计是脆弱的，因为你永远不能在House中多态的更换kitchen和bedroom的行为。

如果这个Kitchen是一个值对象比如：链表，图，用户，邮箱地址，等等，那么只要这个值对象没有指向service对象，我们可以内联创建它们。Service对象是那种最可能被test-double替换的类型，所以你永远都不想把它们放在直接实例化里或者通过静态方法调用。

----------
问题：构造器中有一个部分初始化的对象然后不得不设置它

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// SUT initializes collaborators. This prevents
//   tests and users of Garden from altering them.

class Garden {
  Garden(Gardener joe) {
    joe.setWorkday(new TwelveHourWorkday());
    joe.setBoots(
       new BootsWithMassiveStaticInitBlock());
    this.joe = joe;
  }

  // ...
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Let Guice create the gardener, and have a
//   provider configure it.

class Garden {
  Gardener joe;

  @Inject
  Garden(Gardener joe) {
    this.joe = joe;
  }

  // ...
}

// In the Module configuring Guice.

@Provides
Gardener getGardenerJoe(Workday workday,
    BootsWithMassiveStaticInitBlock badBoots) {
  Gardener joe = new Gardener();
 &nbsp;joe.setWorkday(workday);

  // Ideally, you'll refactor the static init.
  joe.setBoots(badBoots);
  return joe;
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// A test that is very slow, and forced
//   to run the static init block multiple times.

class GardenTest extends TestCase {

  public void testMustUseFullFledgedGardener() {
    Gardener gardener = new Gardener();
    Garden garden = new Garden(gardener);

    new AphidPlague(garden).infect();
    garden.notifyGardenerSickShrubbery();

   &nbsp;assertTrue(gardener.isWorking());

  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// The new tests run quickly and are not
//   dependent on the slow
//   BootsWithMassiveStaticInitBlock

class GardenTest extends TestCase {

  public void testUsesGardenerWithDummies() {
    Gardener gardener = new Gardener();
    gardener.setWorkday(new OneMinuteWorkday());
    // Okay to pass in null, b/c not relevant
    //   in this test.
    gardener.setBoots(null);

    Garden garden = new Garden(gardener);

    new AphidPlague(garden).infect();
    garden.notifyGardenerSickShrubbery();

   &nbsp;assertTrue(gardener.isWorking());
  }
}</pre>
</td>
</tr>
</tbody>
</table>

创建对象图（为Garden创建和配置Gardener collaborators）是一个不同于Garden应该做的职责。当配置和实例化在构造器中混在一起时，对象变得更脆弱以及依赖于具体的对象图结构。代码变得更难以修改，更不可能测试。

* 缺陷：Garden 需要一个 Gardener，但是配置Gardener不应该是 Garden 的职责。
* 缺陷：Garden的单元测试中工作时间在构造器中被具体设置了，这样强制我们使Joe每天工作12小时。这样强制的依赖会导致测试变慢。在单元测试中，你将希望传入一个短一点的工作时间。
* 缺陷：你不能改变这个启动。你可能会想为这个启动使用一个test-double，去避免加载和使用`BootsWithMassiveStaticInitBlock`。（静态初始化块通常是危险和麻烦的，尤其当他们与全局状态交互的时候。）

当你需要初始化时有collaborators，你可以有两个对象。初始化它们，然后将完全初始化的它们传入相关类的构造器中。

------
问题：构造器中违反迪米特法则

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Violates the Law of Demeter
// Brittle because of excessive dependencies
// Mixes object lookup with assignment

class AccountView {
&nbsp; User user;
&nbsp; AccountView() {
&nbsp;&nbsp; user = RPCClient.getInstance().getUser();
&nbsp; }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>class AccountView {
&nbsp; User user;

  @Inject
&nbsp; AccountView(User user) {
&nbsp;   this.user = user;
  }
}

// The User is provided by a GUICE provider
@Provides
User getUser(RPCClient rpcClient) {
  return rpcClient.getUser();
}

// RPCClient is also provided, and it is no longer
//   a JVM Singleton.
@Provides @Singleton
RPCClient getRPCClient() {
  // we removed the JVM Singleton
  //   and have GUICE manage the scope
  return new RPCClient();
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Hard to test because needs real RPCClient
class ACcountViewTest extends TestCase {

  public void testUnfortunatelyWithRealRPC() {
    AccountView view = new AccountView();
    // Shucks! We just had to connect to a real
    //   RPCClient. This test is now slow.

    // ...
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Easy to test with Dependency Injection
class AccountViewTest extends TestCase {

  public void testLightweightAndFlexible() {
    User user = new DummyUser();
    AccountView view = new AccountView(user);
    // Easy to test and fast with test-double
    //   user.

    // ...
  }
}</pre>
</td>
</tr>
</tbody>
</table>

在这个例子中，我们接触了应用的全局状态，得到了一个可控的RPCClient单例。然而我们并不需要这个单例，我们只需要User。第一：我们正在工作（与静态方法相反，这个没有Seams）。第二：这违反了迪米特法则。

* 缺陷：我们不能轻易地拦截 RPCClient.getInstance() ，为测试返回一个模拟的RPCClient。（静态方法是不可截取的，不可模拟的。）
* 缺陷：如果在测试的类不需要RPCCient，为什么我们还不得不为测试模拟 RPCClient？（AccountView 并不一定要在一个字段中使用rpc实例。）我们只需要使用User。
* 缺陷：每个需要构造类AccountView的测试都不得不处理以上这些问题。如果我们为一个测试解决了这个问题，我们不想再为其他测试处理一次。比如，AccountServlet 可能需要 AccountView。因此在AccountServlet我们将不得不成功的操纵这个构造器。

这个改进版的代码中只传入了直接必须的 User collaborator。为了测试，你所需要做的就是创建一个（真的或者test-double）User对象。这导致更灵活的设计和更好的可测试性。

我们使用Guice为我们提供一个从RPCClient得到的User。在单元测试中，我们将不需要Guice，而是直接创建一个User和AccountView。

------
问题：在构造器中创建不必要的第三方对象

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Creating unneeded third party objects,
//   Mixing object construction with logic, &amp;
//   "new" keyword removes a seam for other
//   EngineFactory's to be used in tests.
//   Also ties you to the (slow) file system.

class Car {
  Engine engine;
  Car(File file) {
    String model = readEngineModel(file);
    engine = new EngineFactory().create(model);
  }

  // ...
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Asks for precisely what it needs

class Car {
  Engine engine;

  @Inject
  Car(Engine engine) {
    this.engine = engine;
  }

  // ...
}

// Have a provider in the Module
//   to give you the Engine
@Provides
Engine getEngine(
    EngineFactory engineFactory,
    @EngineModel String model) {
  //
  return engineFactory
      .create(model);
}

// Elsewhere there is a provider to
//   get the factory and model</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// The test exposes the brittleness of the Car
class CarTest extends TestCase {

  public void testNoSeamForFakeEngine() {
    // Aggh! I hate using files in unit tests
    File file = new File("engine.config");
    Car car = new Car(file);

    // I want to test with a fake engine
    //   but I can't since the EngineFactory
    //   only knows how to make real engines.
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Now we can see a flexible, injectible design
class CarTest extends TestCase {

  public void testShowsWeHaveCleanDesign() {
    Engine fakeEngine = new FakeEngine();
    Car car = new Car(fakeEngine);

    // Now testing is easy, with the car taking
    //   exactly what it needs.
  }
}</pre>
</td>
</tr>
</tbody>
</table>

语义上，为了得到一个Car得得到一个EngineFactory去创建自己的engine不太合理。Cars 应当被提供已经准备的 engines，而不是自己去弄明白如何创建engines。你在开往公司的车不应当有一个指向到它工厂的引用。同样，有些构造器接触到并不直接需要的第三方对象，只需要第三方对象创建出来的某些东西。

* 缺陷：当最需要的只是一个Engine时，传入了一个文件。
* 缺陷：创建第三方对象（EngineFactory）以及为这个非注入不可覆写的创建付出代价。你的代码变得很脆弱因为你不能改变这个工厂，你不能决定开始贮存它们，当一个新Car被创建了你不能阻止它运行。
* 缺陷：car知道怎样去构造一个EngineFactory，知道怎样去构建一个engine，这很蠢。（在某种程度上，当这些对象更抽象，我们会容易察觉到自己在犯这个错误）。
* 缺陷：每个需要构造类Car的测试 将不得不处理以上这些问题。虽然我们为一个测试解决了这个问题，我们不想为其他测试再次解决它们。比如给可能需要Car的Garage的一个测试。因此在Garage测试中欧，我们将不得不成功的操纵这个Car构造器。并且我将被迫创建一个新的EngineFactory。
* 缺陷：当这个Car构造器被调用，每个测试将需要一个file的入口。这会很慢，而且阻止测试成为真正的单元测试。

去掉这些第三方对象，在构造器中用简单的变量赋值来替代这些第三方对象的工作。分配一个预配置的变量给构造器中的字段。另外有一个对象（工厂，生成器，或者Guice提供商）做生成该构造器的参数的实际工作。将主功能从对象图构造器的职责中分离开，然后你将得到一个更灵活和易维护的设计。

------
问题：在构造器中直接阅读标志值

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Reading flag values to create collaborators

class PingServer {
  Socket socket;
  PingServer() {
    socket = new Socket(FLAG_PORT.get());
  }

  // ...
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Best solution (although you also could pass
//   in an int of the Socket's port to use)

class PingServer {
  Socket socket;

  @Inject
  PingServer(Socket socket) {
    this.socket = socket;
  }
}

// This uses the FlagBinder to bind Flags to
// the @Named annotation values. Somewhere in
// a Module's configure method:
  new FlagBinder(
      binder().bind(FlagsClassX.class));

// And the method provider for the Socket
  @Provides
  Socket getSocket(@Named("port") int port) {
    // The responsibility of this provider is
    //   to give a fully configured Socket
    //   which may involve more than just "new"
    return new Socket(port);
  }</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// The test is brittle and tied directly to a
//   Flag's static method (global state).

class PingServerTest extends TestCase {
  public void testWithDefaultPort() {
    PingServer server = new PingServer();
    // This looks innocent enough, but really
    //   it forces you to mutate global state
    //   (the flag) to run on another port.
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// The revised code is flexible, and easily
//   tested (without any global state). 

class PingServerTest extends TestCase {
  public void testWithNewPort() {
    int customPort = 1234;
    Socket socket = new Socket(customPort);
    PingServer server = new PingServer(socket);

    // ...
  }
}</pre>
</td>
</tr>
</tbody>
</table>

表面上看起来简单的没有任何参数的构造器，实则有很多依赖。这个API一再欺骗你，假装它很容易创建，实际上PingServer依赖于global state，非常脆弱。

* 缺陷：在你的测试中，为了实例化这个类，你不得不依赖全局变量FLAG_PORT。因为测试顺序的原因你的测试将非常不可靠。
* 缺陷：依赖一个静态的读取标志阻止跑你并发测试。因为并发测试会同时改变这个值，引发失败。
* 缺陷：如果socket需要额外的配置（比如调用setSoTimeout()），由于对象构建在错误的位置于是不可能发生。Socket被创建在PingServer内部，是向后的。它需要在外部创建，在一个唯一的职责是对象图构造的东西里，比如Guice Provider。

PingServer其实需要的是一个socket，而不是一个端口号。我们将不得不用真是的套接口或线程测试因为传入了端口号。如果只是传入一个socket，我们可以在测试的时候创建一个模拟的socket，然后不需要真是的套接口或者线程即可测试。明确的传递端口号移除了对全局状态的依赖，和简化测试。虽然更好的方式是传入一个它根本上需要的socket。

-------------
问题：在构造器中直接读取标识和创建对象

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Branching on flag values to determine state.

class CurlingTeamMember {
  Jersey jersey;

  CurlingTeamMember() {
    if (FLAG_isSuedeJersey.get()) {
      jersey = new SuedeJersey();
    } else {
      jersey = new NylonJersey();
    }
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// We moved the responsibility of the selection
//   of Jerseys into a provider.

class CurlingTeamMember {
  Jersey jersey;

  // Recommended, because responsibilities of
  // Construction/Initialization and whatever
  // this object does outside it's constructor
  // have been separated.
  @Inject
  CurlingTeamMember(Jersey jersey) {
    this.jersey = jersey;
  }
}

// Then use the FlagBinder to bind flags to
//   injectable values. (Inside your Module's
//   configure method)
  new FlagBinder(
      binder().bind(FlagsClassX.class));

// By asking for Provider&lt;SuedeJersey&gt;
//   instead of calling new SuedeJersey
//   you leave the SuedeJersey to be free
//   to ask for its dependencies.
@Provides
Jersey getJersey(
     Provider&lt;SuedeJersey&gt; suedeJerseyProvider,
     Provider&lt;NylonJersey&gt; nylonJerseyProvider,
     @Named('isSuedeJersey') Boolean suede) {
  if (suede) {
   &nbsp;return suedeJerseyProvider.get();
  } else {
    return nylonJerseyProvider.get();
  }
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Testing the CurlingTeamMember is difficult.
//   In fact you can't use any Jersey other
//   than the SuedeJersey or NylonJersey.

class CurlingTeamMemberTest extends TestCase {
  public void testImpossibleToChangeJersey() {
    //  You are forced to use global state.
   &nbsp;// ... Set the flag how you want it
    CurlingTeamMember russ =
        new CurlingTeamMember();

    // Tests are locked in to using one
    //   of the two jerseys above.
 &nbsp;}
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// The code now uses a flexible alternataive:
//   dependency injection.

class CurlingTeamMemberTest extends TestCase {

  public void testWithAnyJersey() {
    // No need to touch the flag
    Jersey jersey = new LightweightJersey();
    CurlingTeamMember russ =
        new CurlingTeamMember(jersey);

    // Tests are free to use any jersey.
 &nbsp;}
}</pre>
</td>
</tr>
</tbody>
</table>

Guice有一个叫做FlagBinder的方法让你以最小的成本移除标志引用，以及以注入值替换标识引用。标识普遍用于改变运行参数，然而我们并不需要直接读取标识的全局状态。

* 缺陷：直接读取标识为了得到一个值直接接触了全局变量。这是不利的，因为全局变量没有脱离出来：其他的测试可以将它设置一个不同的值，或者其他线程会意外的改变它。
* 缺陷：依赖一个标识去直接构建一个不同类型的Jersey。需要实例化CurlingTeamMember的测试将没办法使用Seam去为测试注入一个不同的Jersey collaborators
* 缺陷：CurlingTeamMember的职责很广：除了这个类的主要目的，现在还有配置Jersey。相比之下，传入一个提前配置好的Jersey对象更好。把配置Jersey的职责交给另外一个对象。

使用FlagBinder(一个知道如何去绑定可注入参数的命令行标识的类）将一个类中所有标识加入到Guice的可注入的范围中。

--------
问题：将构造器的工作转移到一个初始化方法中

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// With statics, singletons, and a tricky
//   initialize method this class is brittle.

class VisualVoicemail {
  User user;
  List&lt;Call&gt; calls;

  @Inject
  VisualVoicemail(User user) {
    // Look at me, aren't you proud? I've got
    // an easy constructor, and I use Guice
    this.user = user;
  }

  initialize() {
     Server.readConfigFromFile();
     Server server = Server.getSingleton();
     calls = server.getCallsFor(user);
  }

  // This was tricky, but I think I figured
  // out how to make this testable!
  @VisibleForTesting
  void setCalls(List&lt;Call&gt; calls) {
    this.calls = calls;
  }

  // ...
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Using DI and Guice, this is a
//   superior design.

class VisualVoicemail {
  List&lt;Call&gt; calls;

  VisualVoicemail(List&lt;Call&gt; calls) {
    this.calls = calls;
  }
}

// You'll need a provider to get the calls
@Provides
List&lt;Call&gt; getCalls(Server server,
    @RequestScoped User user) {
  return server.getCallsFor(user);
}

// And a provider for the Server. Guice will
//  let you get rid of the JVM Singleton too.
@Provides @Singleton
Server getServer(ServerConfig config) {
  return new Server(config);
}

@Provides @Singleton
ServerConfig getServerConfig(
    @Named("serverConfigPath") path) {
  return new ServerConfig(new File(path));
}

// Somewhere, in your Module's configure()
//   use the FlagBinder.
  new FlagBinder(binder().bind(
      FlagClassX.class))</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Brittle code exposed through the test

class VisualVoicemailTest extends TestCase {

  public void testExposesBrittleDesign() {
    User dummyUser = new DummyUser();
    VisualVoicemail voicemail =
        new VisualVoicemail(dummyUser);
    voicemail.setCalls(buildListOfTestCalls());

    // Technically this can be tested, as long
    //   as you don't need the Server to have
    //   read the config file. But testing
    //   without testing the initialize()
    //   excludes important behavior.

    // Also, the code is brittle and hard to
    //   later on add new functionalities.
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Dependency Injection exposes your
//   dependencies and allows for seams to
//   inject different collaborators.

class VisualVoicemailTest extends TestCase {

  VisualVoicemail voicemail =
      new VisualVoicemail(
          buildListOfTestCalls());

  // ... now you can test this however you want.
}</pre>
</td>
</tr>
</tbody>
</table>

转移到一个初始化方法中不是解决办法。你需要解耦你的对象，实现单一职责。（这里单一职责就是提供一个配置完全的对象图）

* 缺陷：第一眼看来好像很有效的用到了Guice。对测试而言，VisualVoicemail对象很容易构建。然而代码依旧很脆弱，与几个静态初始化调用绑定在一起。
* 缺陷：这个初始化方法是这个对象有太多职责的一个明显的标志：无论VisualVoicemail需要做什么，都要初始化它的依赖。依赖初始化应当在另外一个类中，然后将所有可以使用的对象传给构造器。
* 缺陷：如果你想要调用初始化方法，在测试中Server.readConfigFromFile()方法是不可截取的。
* 缺陷：这个服务在测试中是不可初始化的。如果你想要用它，你被迫从一个全局单例状态中获取它。如果两个测试并行测试，或者之前的测试有差异的初始化了该服务，全局变量将要刺痛你。
* 缺陷：通常，`@VisibleForTesting`注解是一个信号，说明这个类不是被写来容易测试的。尽管你可以设置调用的方法，这只是以初始化方法的hack来逃避根本问题。

解决这个缺陷，就像其他所有一样，包括移除虚拟机强制的全局状态，和使用依赖注入。


----------

问题：拥有多个构造器，其中一个只是用来测试

<table style="border-color: #888888; border-width: 1px; border-collapse: collapse;" border="1" cellspacing="0" bordercolor="#888888">
<tbody>
<tr>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">Before: Hard to Test</span></strong></td>
<td style="width: 50%;" bgcolor="#000000"><strong><span style="color: #ffffff;">After: Testable and Flexible Design</span></strong></td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Half way easy to construct. The other half
//   expensive to construct. And for collaborators
//   that use the expensive constructor - they
//   become expensive as well.

class VideoPlaylistIndex {
  VideoRepository repo;

  @VisibleForTesting
  VideoPlaylistIndex(
      VideoRepository repo) {
    // Look at me, aren't you proud?
    // An easy constructor for testing!
    this.repo = repo;
 &nbsp;}

  VideoPlaylistIndex() {
    this.repo = new FullLibraryIndex();
  }

  // ...

}

// And a collaborator, that is expensive to build
//   because the hard coded index construction.

class PlaylistGenerator {

  VideoPlaylistIndex index =
      new VideoPlaylistIndex();

  Playlist buildPlaylist(Query q) {
    return index.search(q);
 &nbsp;}
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Easy to construct, and no other objects are
//   harmed by using an expensive constructor.

class VideoPlaylistIndex {
  VideoRepository repo;

  VideoPlaylistIndex(
      VideoRepository repo) {
    // One constructor to rule them all
    this.repo = repo;
 &nbsp;}
}

// And a collaborator, that is now easy to
//   build.

class PlaylistGenerator {
  VideoPlaylistIndex index;

  // pass in with manual DI
  PlaylistGenerator(
      VideoPlaylistIndex index) {
    this.index = index;
  } 

  Playlist buildPlaylist(Query q) {
    return index.search(q);
 &nbsp;}
}</pre>
</td>
</tr>
<tr>
<td style="width: 50%;" bgcolor="#ffc0cb">
<pre>// Testing the VideoPlaylistIndex is easy,
//  but testing the PlaylistGenerator is not!

class PlaylistGeneratorTest extends TestCase {

  public void testBadDesignHasNoSeams() {
    PlaylistGenerator generator =
        new PlaylistGenerator();
    // Doh! Now we're tied to the
    //   VideoPlaylistIndex with the bulky
    //   FullLibraryIndex that will make slow
    //   tests.
  }
}</pre>
</td>
<td style="width: 50%;" bgcolor="#ccffcc">
<pre>// Easy to test when Dependency Injection
//   is used everywhere. 

class PlaylistGeneratorTest extends TestCase {

  public void testFlexibleDesignWithDI() {
    VideoPlaylistIndex fakeIndex =
        new InMemoryVideoPlaylistIndex()
    PlaylistGenerator generator =
        new PlaylistGenerator(fakeIndex);

    // Success! The generator does not care
    //   about the index used during testing
    //   so a fakeIndex is passed in.
  }
}</pre>
</td>
</tr>
</tbody>
</table>

多个构造器，其中有些只用来做测试，暗示着你部分的代码将很难测试。左边，VideoPlaylistIndex很容易测试（你可以传入一个test-double `VideoRepository`）。然后，无论哪个没有任何参数却有着依赖对象的构造器将很难测试。

* 缺陷：`PlaylistGenerator`很难测试，因为`VideoPlaylistIndex`是没有参数的构造器，然后硬编码的使用了`FullLibraryIndex`。你可能不是真的想在测试`PlaylistGenerator`测试`FullLibraryIndex`，然后你不得不。
* 缺陷：通常，`@VisibleForTesting`注解是一个信号，说明这个类不是被写来容易测试的。尽管你可以设置调用的方法，这只是以初始化方法的hack来逃避根本问题。

理想情况下，`PlaylistGenerator`在自己的构造器中请求`VideoPlaylistIndex`，而不是直接创建它自己的依赖。一旦`PlaylistGenerator`请求自己的依赖，就不需要调用没有参数的`VideoPlaylistIndex`构造器，我们就可以删掉它了。我们通常不需要多个构造器。




















