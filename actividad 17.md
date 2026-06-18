# Reporte de Laboratorio: Arquitectura Multi-Nivel (N-Tier) y Patrón MVC en .NET

**Estudiante:** Alejandro
**Institución:** Universidad de San Carlos de Guatemala (USAC)
**Facultad:** Ingeniería  
**Curso:** Introducción a la Programación y Computación 2  

---

## Parte 1: Fundamentación Teórica y Análisis Crítico

### 1. El Tránsito hacia los Sistemas Distribuidos y Multi-Capa

*   **La Limitación del Monolito Local:**  
    Cuando la interfaz de usuario, la lógica de negocio y el almacenamiento de datos residen exclusivamente en una única máquina física aislada, el sistema enfrenta graves problemas de sincronización y escalabilidad. Si múltiples usuarios necesitan acceder o actualizar la información, los datos quedan aislados en silos locales, imposibilitando la consistencia en tiempo real. Además, el escalamiento se vuelve puramente vertical (requiere hardware más costoso en esa única máquina) y un fallo de hardware local inhabilita el sistema por completo (punto único de fallo).

*   **Distinción Crítica (Layers vs. Tiers):**  
    *   **Capas Lógicas (Layers):** Se refieren a la organización e interoperabilidad interna del código fuente dentro de una misma unidad de despliegue. Es una separación puramente lógica y conceptual para ordenar responsabilidades (ej. separar clases de acceso a datos de las clases de negocio).
    *   **Niveles Físicos (Tiers):** Se refieren a la distribución física y geográfica del software en infraestructura de hardware o servidores independientes. Cada *Tier* corre en su propio proceso físico o máquina y se comunica con los demás a través de la red (ej. un servidor web, un servidor de aplicaciones y un cluster de bases de datos).

*   **Responsabilidades en la Arquitectura de 3 Niveles:**  
    *   **Nivel 1: Capa de Presentación (Presentation Tier):** Su misión exclusiva es interactuar con el usuario final, capturar sus peticiones visuales y renderizar los resultados. Tecnologías comunes: HTML5, CSS3, JavaScript (React, Angular) o vistas Razor (`.cshtml`) en el cliente/servidor perimetral.
    *   **Nivel 2: Capa de Aplicación o Negocio (Application Tier):** Actúa como el núcleo inteligente del sistema, donde se ejecutan las reglas de negocio, validaciones complejas, algoritmos y procesamiento de datos. Tecnologías comunes: C# en ASP.NET Core Web API / MVC, Java con Spring Boot, o Node.js ejecutándose en servidores de aplicaciones dedicados.
    *   **Nivel 3: Capa de Datos (Data Tier):** Su única responsabilidad es el almacenamiento persistente, la indexación, la integridad y la recuperación eficiente de la información. Tecnologías comunes: Sistemas Gestores de Bases de Datos (RDBMS) como PostgreSQL, SQL Server, MySQL, o soluciones NoSQL como MongoDB.

*   **Seguridad Perimetral:**  
    Exponer públicamente el puerto de una base de datos (como el 1433 de SQL Server o el 5432 de PostgreSQL) a internet es un error crítico de ingeniería porque reduce la superficie de ataque, permitiendo que cualquier actor malicioso intente ataques de fuerza bruta, escaneo de vulnerabilidades del motor o inyecciones directas. La **buena práctica recomendada** dicta que la base de datos debe estar alojada en una red privada virtual (VPC) o subred privada sin IP pública, permitiendo conexiones *únicamente* desde las IPs específicas de los servidores de la Capa de Aplicación (Nivel 2) mediante reglas estrictas de firewall o grupos de seguridad.

### 2. Desacoplamiento Lógico con el Patrón MVC

*   **La Crisis del Código Espagueti:**  
    Mezclar sentencias SQL, lógica matemática y etiquetas HTML dentro de un solo archivo físico degrada exponencialmente el mantenimiento, ya que cualquier cambio visual menor puede corromper la base de datos o romper la lógica interna de cálculo. Además, vuelve imposibles las pruebas unitarias: no se puede testear una fórmula matemática o una consulta a la base de datos de forma aislada sin levantar o simular toda la interfaz gráfica, lo que destruye la automatización de pruebas de software.

*   **Separación de Preocupaciones (SoC):**  
    *   **Modelo:** Representa el estado de la aplicación, las entidades de datos y las reglas del dominio de negocio. Es completamente independiente; tiene estrictamente prohibido conocer cómo, dónde o a través de qué medios se van a renderizar o mostrar los datos que resguarda.
    *   **Vista:** Se define como una entidad pasiva e inteligente. Es pasiva porque no toma decisiones de negocio ni procesa flujos; simplemente renderiza la información que el controlador le inyecta. Es inteligente porque sabe cómo auto-estructurarse visualmente usando plantillas. Tiene **estrictamente prohibido** contener código de lógica de negocio, cálculos de backend complejos o sentencias de acceso directo a datos.
    *   **Controlador:** Es el intermediario táctico y el director de orquesta. Recibe las peticiones HTTP del cliente, interpreta la intención del usuario, selecciona e interactúa con el Modelo correspondiente para obtener o mutar datos, y finalmente decide qué Vista retornar o a qué ruta redirigir el tráfico.

*   **Métricas de Ingeniería de Software:**  
    El patrón MVC organiza el código de modo que cada componente tenga una única responsabilidad bien definida, lo que eleva la **Alta Cohesión** (cada clase hace una sola cosa y la hace bien). Al mismo tiempo, al comunicarse mediante contratos y flujos estandarizados mediados por el controlador, los componentes no dependen íntimamente de la implementación interna del otro, logrando un **Bajo Acoplamiento** que permite modificar una interfaz visual o cambiar el almacenamiento sin alterar el resto del sistema.

---

## Parte 2: Modelado del Ciclo de Vida y Enrutamiento Semántico

### 1. Mapeo Analítico de URLs

Basado en la plantilla jerárquica por defecto `{controller=Home}/{action=Index}/{id?}`, la resolución del framework es la siguiente:

| URL Entrante del Cliente | Clase Controladora Buscada por el Framework | Método (Acción) Ejecutado | Parámetro Inyectado `id` |
| :--- | :--- | :--- | :--- |
| `https://ingenieria.usac.edu.gt/ControlAcademico/Login` | `ControlAcademicoController` | `Login` | *(Ninguno / Nulo)* |
| `https://ingenieria.usac.edu.gt/Estudiante/Historial/20260123` | `EstudianteController` | `Historial` | `20260123` |
| `https://ingenieria.usac.edu.gt/Asignacion/Detalle/10` | `AsignacionController` | `Detalle` | `10` |
| `https://ingenieria.usac.edu.gt/Home` | `HomeController` | `Index` *(Por defecto)* | *(Ninguno / Opcional)* |

### 2. Diagramación del Flujo Interactivo (Ciclo de Vida del Request)

1.  **Petición del Cliente:** El usuario interactúa con el navegador (hace clic en un botón de envío de datos) y el navegador despacha una petición HTTP (ej. `POST /Estudiante/Registrar`) viajando por la red hacia el servidor perimetral.
2.  **Enrutamiento del Framework:** El motor de enrutamiento de ASP.NET Core intercepta la petición entrante en el middleware, analiza la URL estructurada según sus plantillas de convenciones, localiza la clase `EstudianteController` e instancia el despachador.
3.  **Mediación del Controlador:** El controlador recibe el control del flujo y extrae de forma automática los parámetros provenientes del cuerpo de la petición (Model Binding), realizando una validación estructural perimetral rápida (Skinny Controller).
4.  **Mutación del Modelo / Datos:** El controlador delega la persistencia o lógica de negocio interactuando con el Modelo (o almacén simulado en memoria). El Modelo procesa el nuevo estado, añade el registro y confirma la operación de manera aislada de la interfaz.
5.  **Renderizado dinámico y Respuesta:** El controlador selecciona la vista pertinente o genera el resultado esperado, el motor de vistas de .NET procesa de manera pasiva el código HTML inyectándole los datos limpios y el servidor envía de regreso al cliente una respuesta HTTP limpia con el código HTML dinámico o el código de estado (`201 Created`).

---

## Parte 5: Referencias Bibliográficas

*   Facultad de Ingeniería, USAC. (2026). Sesión 11: Modelado Base y Arquitecturas de Despliegue. Evolución de Sistemas Distribuidos, Fundamentos del Modelo Cliente-Servidor y Diseño Físico Multi-Capas (N-Tier). Laboratorio del curso Introducción a la Programación y Computación 2. Guatemala.
*   Facultad de Ingeniería, USAC. (2026). Sesión 12: Arquitectura y Componentes del Patrón MVC. Desacoplamiento Lógico de Software, Ciclo de Vida de las Peticiones y Enrutamiento en Aplicaciones Interactivas Modernas. Laboratorio del curso Introducción a la Programación y Computación 2. Guatemala.