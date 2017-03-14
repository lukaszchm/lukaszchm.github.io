---
layout: post
title:  "Dependency injection w Service Fabric"
date:   2017-03-09 15:03:43 +1000
categories: servicefabric IoC
---
Od jakiegoś czasu pracuję dużo z Microsoft Service Fabric. 
Kiedy zaczynałem, jedną z pierwszych rzeczy jaką chciałem mieć gotową w swoim zbiorze narzędzi był mechanizm Dependency Injection. 
Zawsze, kiedy zaczynam pracować z nowym frameworkiem czy nową platformą, lubię możliwie jak najwcześniej mieć 
zaimplementowany i gotowy to użycia właśnie ten wzorzec. Kiedy zacznie się używać IoC to ciężko jest potem pisać kod bez niego.
Wywołuje się konstruktory lub otrzymuje obiekty za pomocą fabryki czy nawet prostego singltona, ale ma się to wrażenie, że nie 
tak powinno to wyglądać. Od razu pojawiają się pytania w stylu: „Co jeśli zmieni się sytuacja i ten komponent będzie musiał 
być zastąpiony przez inną implementacją?”; „Jak ja przetestuję kiedykolwiek ten kod, przecież żeby go uruchomić potrzebuję mieć 
skonfigurowaną całą aplikację?”. No i oczywiście używanie IoC jest zwyczajnie wygodne. 

Service Fabric nie dostarcza żadnego mechanizmu IoC „z pudełka”. Na szczęście nie rzuca nam też specjalnie kłód pod nogi, 
żeby skonfigurować jakikolwiek z dziesiątków dostępnych na rynku. Skoro o tym mowa, zatrzymajmy się chwilę nad wyborem kontenera 
użytego w tym przykładzie. Przez znacząca większość mojej praktyki jako programista używałem kontenera Castle Windsor. 
Skoro jednak mam zaimplementować IoC na nowej dla mnie platformie, dlaczego by nie spróbować czegoś nowego. Nie to żebym miał coś 
przeciwko Castlowi. To świetny kontener, ma duże możliwości i wtyczki integrujące go z wieloma platformami. Postanowiłem jednak go 
zmienić trochę dla samej satysfakcji zabawy z nowym wirtualnym gadżetem.

### Kontener

Zacząłem więc poszukiwania nowego kontenera IoC dla moich aplikacji. Chyba najpopularniejszy w moich kręgach znajomych jest 
obecnie Autofac i Ninject. Fajnie byłoby też uwzględnić kilka innych produktów w wyborze. Tak oto szukając inspiracji znalazłem 
ten szalenie interesujący [wpis][ioc-benchamrk]. Po analizie wyników Daniela mój wybór padł na [DryIoC][dry-ioc]. 
Jest to bowiem jeden z najszybszych kontenerów, a przy tym posiada pełną gamę funkcjonalności. 
Przynajmniej tych, które kiedykolwiek miałem okazję użyć. 

### Jak to zaimplementować

Wspomniałem, że Service Fabric nie rzuca nam specjalnie kłód pod nogi, jeśli chcemy zaimplementować IoC. 
Nie jest jednak tak jak przyzwyczaił nas ASP.NET, że wystarczy kilka linijek kontener jest podpięty i działa. 
Jest jeden drobny szczegół, który trzeba obejść i to właściwie o tym ma być ten wpis. 

W aplikacji Service Fabric nasz kod będzie uruchamiany w ramach aktorów lub usług Reliable Services. 
Oczywiście pomijając usługi typu Guest Executable, które oczywiście są czymkolwiek i Service Fabric jedynie jest je orkiestruje. 

W pierwszej kolejności przyjrzyjmy się aktorom. Aktor jest dla nas głównym korzeniem warstwy aplikacyjnej. 
To w nim będziemy rozwiązywać pierwsze zależności za pomocą kontenera. Właściwie to nie my będziemy rozwiązywać, 
a zależności będą rozwiązywane. W końcu to odwrócenie sterowania. Musimy w takim razie zmusić kontener do rozwiązania 
zależności zdefiniowanych w naszym aktorze. Zrobimy to w klasyczny sposób pozwalając kontenerowi wywołać dynamicznie 
konstruktor klasy. Musimy w tym celu nieco zmodyfikować kod rejestracji typu aktora. 

{% highlight c# linenos %}
var container = new Container()
    .WithActorFactories();
 
container.RegisterMany(new [] { typeof(Program).Assembly}, 
    type=>type.IsAssignableTo(typeof(IoCEnabledActorBase)));
 
ActorRuntime.RegisterActorAsync<DeviceSensor>(
    (context, typeInformation) =>
    new ActorService(context, typeInformation, 
        container.ActorFactory<DeviceSensor>
    )).GetAwaiter().GetResult();
 
Thread.Sleep(Timeout.Infinite);

{% endhighlight %}

Linie 1-5 to konfiguracja kontenera i rejestracja typu aktora. Omówię to za chwilę. 
Linie 6-10 to rejestracja typu aktora w infrastrukturze Service Fabric. 
Aby ręcznie stworzyć aktora, a w naszym przypadku pobrać z kontenera, musimy po pierwsze 
zarejestrować własną fabrykę tworzenia usługi i w tej usłudze zdefiniować własną fabrykę dla aktorów. 
Fabryka aktorów to delegat według następującej sygnatury: *Func<ActorService, ActorId, ActorBase>*.  
W powyższym przykładzie ta fabryka jest zaimplementowana w innej klasie dla reużywalności i przejrzystości. 

{% highlight c# linenos %}
public static TActor ActorFactory<TActor>(this IResolver container, 
    ActorService actorService, ActorId actorId)
    where TActor : IoCEnabledActorBase
{
    var factory = container.Resolve<IActorInstanceFactory<TActor>>();
    factory.RegisterServiceAndActorId(actorService, actorId);
    return factory.CreateActor();
}

{% endhighlight %}

Powyższa metoda zostanie wywołana za każdym razem, kiedy Service Fabric poczuje potrzebę utworzenia 
nowej instancji aktora. Będzie to miało miejsce za każdym razem kiedy następować będzie aktywacja 
nieaktywnego aktora. Metoda dostaje dwa parametry i oba są wymaganymi parametrami dla konstruktora bazowej 
klasy aktora. My chcemy przecież nie wywoływać jawnie tego konstruktora. Chcemy żeby to kontener nas wyręczył. 
Musimy więc w jakiś sposów przekazać dwie wartości do procesu rozwiazywania komponentu naszego aktora. 
Wyobrażam sobie, że najprosciej byłoby, gdyby metoda Resolve kontenera poza typem do rozwiazania przyjęłaby 
właśnie te dwie wartości i przekazała je do konstruktora. Wszystkie inne zależności byłby rozwiazane natomiast 
z kontenera. Niestety DryIoC nie ma takiej funkcji. Castly również tego nie potrafił. Nie wiem też czy 
jakikolwiek kontener ma taką funkcojnalność. Spróbujmy sobie poradzić z tym co mamy. 

Do osiągnięcia celu potrzebujemy dwóch tymczasowych komponentów. Będą to *ActorActivationContext* oraz generyczny *ActorInstanceFactory*. 
![Class Diagram][class-diagram]
Interakcja pomiędzy komponentami wygląda następująco:
![Sequence Diagram][sequence-diagram]

Fabryka wywołana przez infrastrukturę Service Fabric zaczyna od pobrania z kontenera komponentu 
*ActorInstanceFactory*. Ten zależny jest natomiast od ActorActivationContext oraz od obiektu 
*Lazy<Actor>*. Oba są pobierane z kontenera. Dalej, fabryka wywołuje metodę RegisterServiceAndActorId.
Powoduje to zapisanie dwóch wartości przekazanych do fabryki w obiekcie ActorActivationContext.
Ostatnim krokiem jest wywołanie metody CreateActor. Metoda ta materializuje obiekt Lazy<Actor>.
Wtedy dopiero wywoływany jest właściwy konstruktor naszego aktora. Sam aktor zależy od komponentu 
*ActorActivationContext* w którym przed chwilą zapisaliśmy nasze dwie wartości wymagane przez 
konstruktor bazowy. Takie zachowanie obiektu Lazy<Actor> to jedna z funkcjonalności [DryIoC][dry-ioc-reuse] .
W Castle moglibyśmy osiągnąć to samo rejestrując komponent jako [fabrykę][castle-typed-factory]. Na diagramie sekwencji 
widać jeszcze wywołanie metody *RegisterActorIntance*. W kontekście tego artykułu nie ma to znaczenia. 
Postaram się kiedyś rozwinąć ten pomysł w osobnym wpisie. 

Żeby cała koncepcja zadziałała potrzebujemy jeszcze jednego elementu. Wróćmy do pierwszej części 
kodu rejestracji typu aktora w Service Fabric. Najpierw tworzymy kontener i natychmiast rejestrujemy 
w nim nasze pomocnicze komponenty. 

{% highlight c# linenos %}
public static IContainer WithActorFactories(this IContainer container)
{
    container.RegisterMany<ActorActivationContext>(
        Reuse.InResolutionScope,
        serviceTypeCondition: type=>type.IsInterface);
 
    container.Register(typeof(IActorInstanceFactory<>), 
        typeof(ActorInstanceFactory<>), 
        Reuse.InResolutionScope);
 
    return container;
}
{% endhighlight %}

Jest to typowa rejestracja komponentu w DryIoC i nie chciałbym w tym miejscu przepisywać dokumentacji 
tego produktu. Warto jednak zwrócić uwagę, że oba komponenty rejestrowane są w trybie *Reuse.InResolutionScope*.
W Castly podobny efekt otrzymalibyśmy wykorzystując *lifestyle* typu [bound][castle-bound-filestyle]. 
Ta polityka reużycia komponentu powoduje, że wszędzie w ramach jednego wywołania metody Resolve na kontenerze 
instancja obiektu będzie reużyta. Każde kolejne wywołanie metody Resove spowoduje utworzenie nowej instancji. 
Ponadto działa to tak jak oczekiwalibyśmy również przy użyciu obiektu Lazy. Dokładne zachowanie kontenera 
w tym scenariuszu obrazuje poniższy test.

{% highlight c# linenos %}
[TestClass]
public class DryiocTests
{
    public class Component
    {
        public Component(CreationContext context)
        {
            Parameter = context.Parameter;
        }
 
        public object Parameter { get; }
    }
 
    public class Factory
    {
        private readonly CreationContext _context;
        private readonly Lazy<Component> _lazyComponent;
        public Factory(CreationContext context, Lazy<Component> lazyComponent)
        {
            _context = context;
            _lazyComponent = lazyComponent;
        }
 
        public Component CreateComponent()
        {
            return _lazyComponent.Value;
        }
 
        public object Parameter
        {
            set { _context.Parameter = value; }
        }
    }
 
    public class CreationContext
    {
        public object Parameter { get; set; }
    }
 
    [TestMethod]
    public void test_lazy_resolve_and_resolutionscope_reuse()
    {
        var container = new Container();
        container.Register<Component>(Reuse.InResolutionScope);
        container.Register<CreationContext>(Reuse.InResolutionScope);
        container.Register<Factory>(Reuse.Transient);
 
        var object1 = new object();
        var factory1 = container.Resolve<Factory>();
        factory1.Parameter = object1;
 
        var object2 = new object();
        var factory2 = container.Resolve<Factory>();
        factory2.Parameter = object2;
 
 
        var component1 = factory1.CreateComponent();
        var component2 = factory2.CreateComponent();
 
        Assert.AreSame(component1.Parameter, object1);
        Assert.AreNotSame(component1.Parameter, component2.Parameter);
    }
}
{% endhighlight %}

Tym sposobem za każdym razem, kiedy będziemy potrzebowali wykorzystać nowy komponent w naszym aktorze, wystarczy dodać jego interfejs do konstruktora, zapisać go w polu i gotowe. 

Niestety nie jest to rozwiązanie wymarzone. Moim głównym zarzutem dla takiego podejścia jest fakt, że możemy łatwo podpiąć do aktora nowe zależności, ale wciąż jesteśmy zależni od infrastruktury Service Fabric. Nie ma łatwego sposobu, aby przetestować kod aktora oddzielnie od menagera stanu czy kontekstu. Pierwotnie moim celem było stworzenie infrastruktury w kodzie tak, aby klasa implementująca interfejs aktora nie dziedziczyła po klasie *ActorBase*. Mielibyśmy wtedy klasę złożoną z czystej logiki bez żadnych zależności od infrastruktury. Pozwoliłoby to prowadzić prawdziwe *Test Driven Development*. Próbowałem osiągnąć to używając *Castle Dynamic Proxy*, ale niestety Service Fabric jest zbyt czuły i nie pozwolił mi w żaden sposób przemycić dynamicznie generowanej klasy jako typ aktora. Na tą chwilę się poddaję i w swoich aplikacjach będę stosował DryIoC tak jak pokazuje to ten wpis. 

Przykładowy projekt spinający wszystko w całość dostępny jest na [Githubie][github-project].

[ioc-benchamrk]: http://www.palmmedia.de/blog/2011/8/30/ioc-container-benchmark-performance-comparison
[dry-ioc]: https://bitbucket.org/dadhi/dryioc
[dry-ioc-reuse]: https://bitbucket.org/dadhi/dryioc/wiki/ReuseAndScopes
[castle-typed-factory]: https://github.com/castleproject/Windsor/blob/master/docs/typed-factory-facility-interface-based.md
[castle-bound-filestyle]: https://github.com/castleproject/Windsor/blob/master/docs/lifestyles.md#user-content-bound
[class-diagram]: /assets/sf-ioc/ClassDiagram.png
[sequence-diagram]: /assets/sf-ioc/Sequence.png
[github-project]: https://github.com/lukaszchm/ServiceFabric-Utilities