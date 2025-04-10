# 🧯 Patrón Circuit Breaker

El patrón **Circuit Breaker** (cortacircuitos) es una técnica de resiliencia ampliamente usada en arquitecturas distribuidas y microservicios. Su objetivo principal es **evitar fallas en cascada**, proteger los servicios ante errores y mejorar la disponibilidad general del sistema.

---

## 📌 ¿Qué es?

Circuit Breaker es un **patrón de diseño estructural** que actúa como un interruptor entre dos partes de una aplicación, controlando las llamadas a servicios externos o componentes que podrían fallar. Si un servicio empieza a fallar repetidamente, el circuito se "abre" y bloquea temporalmente las llamadas a dicho servicio para **evitar que se propaguen los fallos**.

---

## 🛠️ ¿Para qué sirve?

- **Prevenir errores en cascada** en arquitecturas distribuidas.
- **Reducir la carga** en servicios que ya están fallando.
- **Evitar bloqueos** o tiempos de espera innecesarios.
- **Aumentar la resiliencia** del sistema.
- **Proporcionar respuestas de respaldo (fallbacks)** ante fallas.

---

## ⚙️ ¿Cómo funciona?

Circuit Breaker utiliza **tres estados principales**:

1. **🔒 Cerrado (Closed):**
    - El sistema opera con normalidad.
    - Se registran los resultados de las llamadas (éxito o falla).
    - Si el número o el porcentaje de fallos excede un umbral, se abre el circuito.

2. **🔓 Abierto (Open):**
    - Las llamadas al servicio fallan inmediatamente.
    - No se realizan intentos de conexión.
    - Se puede devolver una respuesta alternativa o error controlado.
    - El sistema espera un tiempo determinado antes de pasar al siguiente estado.

3. **🚧 Semiabierto (Half-Open):**
    - Se permiten unas pocas llamadas "de prueba".
    - Si estas llamadas son exitosas, el circuito vuelve a cerrado.
    - Si fallan, se vuelve a abrir.

![imagen](/images/img.png)

---

## 🧩 ¿Qué tipos hay?

1. **Basado en conteo de errores:**
    - Se abre si se detecta un número específico de errores consecutivos.

2. **Basado en tasa de errores:**
    - Se abre si el porcentaje de errores dentro de una ventana de tiempo supera un umbral.

3. **Basado en tiempo de respuesta (timeout):**
    - Se activa cuando el tiempo de respuesta de un servicio es demasiado alto.

4. **Combinado o personalizado:**
    - Se puede configurar con múltiples criterios, como conteo, porcentaje y latencia.

---

## 📅 ¿Cuándo usarlos?

- Cuando se llaman **servicios remotos o APIs de terceros**.
- En **microservicios** donde los componentes dependen unos de otros.
- Cuando es crucial **mantener la disponibilidad** del sistema.
- En **ambientes donde no se puede tolerar alta latencia**.
- Si un servicio crítico **puede degradarse bajo carga o fallo**.

---

## ✅ Beneficios

- Previene sobrecargas y fallos en cadena.
- Mejora la experiencia del usuario al evitar tiempos de espera excesivos.
- Proporciona mayor control sobre el comportamiento ante fallos.
- Permite implementar lógica de fallback personalizada.

---

## Ejemplo

pom.yml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.0.2</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

applicacion.properties
```properties
server.port=8080
server.servlet.context-path=/circuit-breaker-demo

jackson.serialization.indent_output=true

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.health.circuitbreakers.enabled=true

resilience4j.circuitbreaker.configs.default.registerHealthIndicator=true
resilience4j.circuitbreaker.configs.default.slidingWindowSize=10
resilience4j.circuitbreaker.configs.default.minimumNumberOfCalls=5
resilience4j.circuitbreaker.configs.default.permittedNumberOfCallsInHalfOpenState=3
resilience4j.circuitbreaker.configs.default.automaticTransitionFromOpenToHalfOpenEnabled=true
resilience4j.circuitbreaker.configs.default.waitDurationInOpenState=5s
resilience4j.circuitbreaker.configs.default.failureRateThreshold=50
resilience4j.circuitbreaker.configs.default.eventConsumerBufferSize=10
```

PaymentManager.java
```java
@Component
public class PaymentManager implements IPaymentManager {


    private RestTemplate restTemplate;

    @Autowired
    public PaymentManager(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public String processPayment() throws Exception {
        try {
            restTemplate.getForEntity("https://www.google.con", String.class);
            return "Pago procesado correctamente";
        } catch (Exception ex) {
            throw ex;
        }


    }
}
```

PaymentService.java
```java
@Service
public class PaymentService implements IPaymentService {

    private IPaymentManager paymentManager;

    @Autowired
    public PaymentService(IPaymentManager paymentManager) {
        this.paymentManager = paymentManager;
    }

    @Override
    public String processPayment() throws Exception{
        return paymentManager.processPayment();
    }
}
```

PaymentController.java
```java
@RestController
public class PaymentController {

    private final IPaymentService paymentService;

    @Autowired
    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @GetMapping("/processPayment")
    @CircuitBreaker(name = "processPayment", fallbackMethod = "fallbackMethod")
    public String processPayment() throws Exception {
        return paymentService.processPayment();
    }



    public String fallbackMethod(Throwable throwable) {
        return "Lo sentimos, actualmente estamos experimentando dificultades técnicas para procesar pagos en línea. Por favor, inténtalo de nuevo más tarde. Agradecemos tu paciencia y comprensión.";
    }
}
```

- Inicializamos el circuit breaker (http://localhost:8080/circuit-breaker-demo/actuator/circuitbreakers)
  ![imagen1](/images/img1.png)

- Realizamos fallos (http://localhost:8080/circuit-breaker-demo/processPayment)
  ![imagen3](/images/img3.png)

- Observamos que cambia el estado (http://localhost:8080/circuit-breaker-demo/actuator/circuitbreakers)
  ![imagen2](/images/img2.png)

## 🧠 Conclusión

Circuit Breaker es esencial en sistemas resilientes. Al evitar que una parte fallida afecte al resto del sistema, este patrón permite una recuperación más rápida y una experiencia más predecible para el usuario. En combinación con otras técnicas como retries, timeouts y fallbacks, es una herramienta poderosa para diseñar sistemas robustos y tolerantes a fallos.


## 📌 **Referencias**

- [Documentación oficial de patrones de diseño](https://refactoring.guru/es/design-patterns/strategy)
- [Ejemplo en GitHub](https://github.com/borispacex/design-pattern-strategy)
