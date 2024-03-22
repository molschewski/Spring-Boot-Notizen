In diesem Dokument trage ich meine Gedanken zu Spring Boot zusammen. Quellen sind "Spring Boot 3 und Spring Framework 6" von Christian Ullenboom und natürlich die umfangreiche Dokumentation von Spring bzw. Spring Boot im Netz. Und wenn alle Stricke reißen, gibt es immer noch stackoverflow.

# Grundlagen (Über was reden wir hier eigentlich?)
Spring ist ein Framework, um Java Enterprise Anwendungen zu entwickeln. Es ist modular aufgebaut, weitgehend konfigurierbar, und flexibel einsetzbar. Das Herzstück von Spring ist der IoC Container, wobei IoC für _Inversion of Control_ steht. Der Container bietet Mechanismen für die Konfiguration und für Dependency Injection.

Die unfassende Konfigrierbarkeit von Spring erschwert den Einstieg, es muß viel Arbeit investiert werden, damit die Anfangskonfiguration steht.  Um den Einstieg zu erleichtern, ist Spring Boot entstanden. Das ist ein Modul, das auf Spring aufsetzt. Es bietet sinnvolle Vorgaben, und verfolgt den Ansatz "Konvention vor Konfiguration".

## Der Container und die Beans
Container und Beans gehen Hand in Hand. Beans sind Java-Objekte, die über einen speziellen Mechanismus initalisiert und verwaltet werden. Dieser Mechanismus wird von dem Container bereitgestellt.

Die Beans sind Objekte der Anwendung. Typischerweise sind es Objekte wie Repositories, Data Access Objects, Web Controller und ähnliches. Die eigentlichen Domain Objects wie z. B. eine Rechnung in einer Buchhaltungssoftware werden nicht über Beans abgebildet. Das ist Aufgabe der Business Logic.

Mit dem Start des Containers werden die Beans initalisiert. Um die erzeugten Beans in unserem Programm zu verwenden, bietet Spring Boot die Mechanismen der _Invasion of Control_ und der _Dependency Injection_ an. In einem herkömmlichen Programm werden die Objekte bei Bedarf über `new` oder über eine Fabrikmethode erzeugt. Bei der IoC wird diese Verantwortung vom Programm an Spring Boot abgegeben. Dies ermöglicht, vergleichbar der Fabrikmethode, eine Konfiguration zur Laufzeit. Für die Verwendung müßen die Referenzen zu den Beans an den entsprechenden Programmstellen zu Verfügung stehen. Dies geschieht über die Dependency Injection, die Abhängigkeiten zu den Beans werden injiziert.

### Der Container
Der Container ist durch das Interface `org.springframework.context.ApplicationContext` ansprechbar. Es ermöglicht das Instanzieren, Konfigurieren, und Assemblieren der Beans. Die dafür nötigen Instruktionen sind als Metadaten angegeben. Das können XML-Dateien sein, diese werden wir erst einmal nicht im Detail betrachten. Die beiden anderen Formen sind Java Annotations oder Java Code.

### Die Beans
Beans leben in dem Container. Jede Bean hat Metadaten, die aus der Konfiguration gelesen werden. Diese Daten lassen sich in vier Gruppen einteilen.

* Der Name. Der Name ist innerhalb des Containers eindeutig, meistens ist es der Name der Klasse, die die Bean implementiert.
* Das Verhalten. Wie verhält sich die Bean innerhalb des Containers, das umfasst Elemente wie scope, lifecycle callbacks usw.
* Referenzen. Diese Referenzen weisen auf andere Beans, sie werden auch collaborators oder dependencies genannt.
* Alles Andere. Das kann z. B. die maximale Nummer von Verbindungen sein, die von einer entsprechenden Bean gemanagt werden.

# Das Erzeugen und Finden der Beans
Um Beans zu erzeugen, gibt es zwei Möglichkeiten. Entweder werden sie in einer XML-Datei definiert, oder es werden Annotationen verwendet. Die XML-Dateien werden hier wieder ignoriert. Bei der Verwendung von Annotationen werden die Kandidaten für die Beans gefunden, indem der `classpath` gescannt wird. Wenn eine class bestimmte Filterkriterien erfüllt, und mit der passenden Annotation ausgezeichnet ist, wird sie ein Kandidat für eine Bean.

Die generische Auszeichnung für eine von Spring gemanagte Komponente ist `@Component`. Die damit markierte Klasse Wird auch als "Stereotyp" bezeichnet. Es gibt drei spezialisierte Auszeichnungen, die `@Component` verfeinern.
* `@Controller`
* `@Repository`
* `@Service`
Diese Unterscheidung ist heute mehr für die Übersichtlichkeit, in zukünftigen Versionen von Spring können die verschiedenen Annotationen auch unterschiedliches Verhalten aufweisen.

Die Spezialisierung geschieht nicht über Ableitung oder Vererbung. Stattdessen werden Meta-Annotationen verwendet. Eine Meta-Annotation ist eine Annotation, die in einer anderen Annotation verwendet werden kann. Eine Beispiel ist die Annotation `@Service`, sie enthält die Annotation `@Component`. Dies wird bei einem Blick auf die Definition deutlich.

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Component
    public @interface Service {

	    // ...
    }

Eine Annotation kann mehrere Meta-Annotationen beinhalten, Attribute einer Meta-Annotation können überschrieben werden.Genaueres dazu findet sich in der Spring-Dokumentation.

## Wer suchet ...
Die Bean-Kandidaten werden standardmäßig entlang des `classpath` des Projektes gesucht. Die Suche beginnt mit dem Aufuf der Methode `run(...)` der SpringApplication. Der SpringApplication wird eine Konfiguration übergeben, diese ist entweder eine `@Component` oder eine `@Configuration`. Diese Übergabe kann entweder bei der Konstruktion der SpringApplication oder bei dem Aufruf von `run()` erfolgen.

    package com.example.demo;
    
    import org.springframework.boot.SpringApplication;
    
    @SpringBootApplication
    public class MyApplication {

        public static void main(String[] args) {
                SpringApplication.run(MyApplication.class, args);
        }
    }

Hier wird die eigene Hauptklasse als Konfiguration verwendet. Dies ist möglich, da die Annotation `@SpringBootApplication` auf die Annotationen `@SpringBootConfiguration`, `@EnableAutoConfiguration`, und `@ComponentScan` verweist. `@SpringBootConfiguration` ist eine Spezialisierung von `@Configuration`.

Spring empfiehlt, die Hauptklasse hierarchisch über den anderen Klassen anzuordnen. Die Annotation `@SpringBootApplication` wird an diese Klasse geschrieben, damit ist implizit eine Basis für die Suche geschaffen. Als Beispiel geben sie die folgende Struktur an.

    com
     +- example
         +- myapplication
             +- MyApplication.java
             |
             +- customer
             |   +- Customer.java
             |   +- CustomerController.java
             |   +- CustomerService.java
             |   +- CustomerRepository.java
             |
             +- order
                 +- Order.java
                 +- OrderController.java
                 +- OrderService.java
                 +- OrderRepository.java


Die Kombination von `@ComponentScan` und `@Configuration` an der Klasse führt dazu, daß die Suche in dem Paket beginnt, in dem die Klasse liegt. In unserem Fall ist das `com.example.demo`. Die Suche umfasst auch alle Unterpakete. Sollen auch andere Pakete durchsucht werden, können sie mit der Annotation `@ComponentScan` angegeben werden. Durch das Setzen des Wertes wird der Automatismus von SprungBoot abgeschaltet, es wird nicht mehr automatisch in dem Paket mit der Hauptklasse gesucht.  Es kann aber natürlich in die Suchliste aufgenommen werden. Dies sieht in der Praxis z. B. a so aus

    @Configuration
    @ComponentScan("com.example.demo", "com.example.net")

Die Argumente können auch in einem Array angegeben werden.

Die Paketnamen werden als Strings angegeben, was immer etwas problematisch ist. Eine bessere Möglichkeit ist es, auf Klassen in den Paketen zu verweisen. Um Probleme beim Refactoring zu vermeiden, ist die Empfehlung, ein leeres Interface in den Paketen anzulegen. In jedem Paket wird ein Interface nach folgendem Muster angelegt:

    packageg com.example.net;
    interface NetModule{}

Die Konfiguration für das Scannen sieht dann so aus:

    @Configuration
    @ComponentScan(basePackageClasses = {BaseModule.class, NetModule.class})

### Filter

Es gibt zwei Arten von Filtern, `includeFilters` und `excludeFilters`, die bestimmte Typen entweder aufnehmen oder ausschliessen. Filter finden auch Typen, die nicht die Annotation `@Component` haben, diese Typen werden dann als Bean betrachtet. Es werden verschiedene Filter unterstützt, gefiltert werden kann auf

* annotation (default)
* assignable
* aspectj
* regex
* custom

Ein Beispiel, in dem nur nach Annotationen `@FooBar` gesucht wird:

    @Configuration
    @ComponentScan(
        useDefaultFilters = false,
        includeFilters = @ComponentScan.Filter(
            type = FilterType.ANNOTATION // Der Default, kann auch entfallen
            classes = {FooBar.class} )
    )

Durch das Setzen von `useDefaultFilters` auf `false` wird das automatische Scannen abgeschaltet, es werden nur Typen mit der Annotation `@FooBar` gefunden.

### Lazy Initalsation

# Nutze die Beans

Beans können nur in Komponenten injiziert werden, die ebenfalls von Spring gemanagt werden.
Um die Beans nutzbar zu machen, wird der Container beim Start das _Wiring_ durchführen. Die Beans werden initalisiert, die Abhängigkeiten aufgelöst, und an den entsprechenden Verwendungsstellen injiziert.  Um diese Stellen zu kennzeichnen, gibt es drei Möglichkeiten.

## Constructor Injection
Die gewünschte Bean wird als Argument in den Constructor geschrieben. Dabei kann die Constructor Injection eine finale Variable füllen, was mit den beiden anderen Methoden nicht möglich ist.

## Setter Injection
Die Bean wird über einen Setter injiziert. Dabei ist der Methodenname frei wählbar. Spring Boot kann nicht jeden Setter prüfen, deshalb wird er mit `@Autowired` annotiert.

    @Autowired
	public void setFooComp(FooComp foo) {
		this.foo = foo;
	}

## Field Injection
Bei der Field injection wird eine Objektvariable mit `@Autowired` annotiert, dies führt dazu, daß die Variable beim Wiring automatisch gesetzt wird.

    @Autowired private BarComp bar;

# Start

In Spring Boot gibt es die `Runner`, die nach dem Start des Containers ausgeführt werden. `ApplicationRunner`.

# Quellen

https://docs.spring.io/spring-framework/reference/overview.html
https://docs.spring.io/spring-boot/docs/3.2.0/reference/html/
https://stackoverflow.com/questions/tagged/spring-boot
