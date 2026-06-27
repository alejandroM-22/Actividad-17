# Actividad de Laboratorio: Interoperabilidad y Carga Masiva de Datos

**Estudiante:** Alejandro  
**Institución:** Universidad de San Carlos de Guatemala (USAC)  
**Curso:** Introducción a la Programación y Computación 2  

---

## Parte 1: Evaluación Conceptual y Buenas Prácticas

### Formatos de Intercambio: Tabla Comparativa

| Formato | Ventajas | Desventajas |
| :--- | :--- | :--- |
| **CSV** | • Formato extremadamente ligero y de bajo consumo de ancho de banda.<br>• Estructura simple, fácil de leer y generar.<br>• Ideal para la transferencia masiva de datos tabulares limpios. | • Carece de un esquema nativo para definir tipos de datos o jerarquías complejas.<br>• Vulnerable a fallos de parseo si los datos contienen comas o saltos de línea sin escapar. |
| **XML** | • Permite una estructura jerárquica y compleja de datos mediante el uso de etiquetas personalizadas.<br>• Soporta validación estricta de esquemas (XSD) y metadatos.<br>• Altamente legible por humanos y compatible con sistemas legados. | • Alta verbosidad que incrementa significativamente el tamaño del archivo y el consumo de red.<br>• El proceso de parseo consume más recursos de procesamiento (CPU y memoria) en comparación con CSV o JSON. |

### 1. Diferenciación de Procesos: Serialización y Deserialización

Utilizando la librería nativa `System.Text.Json`, la diferencia técnica radica en la dirección de la transformación de los datos y el tipo de recurso que se manipula:

* **Serialización (`JsonSerializer.Serialize`):** Es el proceso técnico de transformar un objeto en memoria (una instancia de una clase de C#) a un formato de texto plano estructurado en una cadena de caracteres (o flujo de bytes) bajo el estándar JSON. Se utiliza principalmente para preparar los datos antes de enviarlos a través de la red o almacenarlos en un archivo.
* **Deserialización (`JsonSerializer.Deserialize`):** Es el proceso inverso, donde se toma una cadena de texto o un flujo de datos en formato JSON y se parsea para reconstruir dinámicamente un objeto equivalente en la memoria de la aplicación (fuertemente tipado en C#). Esto permite que el sistema web interactúe directamente con las propiedades y tipos nativos del lenguaje.

### 2. El Antipatrón del Rendimiento: Problema "N+1" y Batching

* **El Problema "N+1":** Consiste en un error crítico de rendimiento que ocurre cuando, durante la lectura de un archivo masivo (de $N$ registros), el sistema realiza una consulta o inserción individual en la base de datos por cada línea leída. Esto genera $1$ consulta inicial para abrir el proceso y $N$ llamadas adicionales a la base de datos a lo largo del ciclo, provocando una degradación masiva del rendimiento debido a la latencia de red recurrente (I/O) y el bloqueo de conexiones.
* **Estrategia de Optimización por Lotes (Batching):** Para solucionarlo, se implementa la acumulación intermedia de los datos. En lugar de impactar la base de datos línea por línea, los registros se mapean y se guardan temporalmente en una lista en memoria. Al finalizar el procesamiento del archivo (o al alcanzar un tamaño de lote óptimo), se ejecuta una única operación de inserción en bloque (por ejemplo, mediante `AddRange()` en Entity Framework) y se confirma la transacción con una sola llamada asíncrona (`SaveChangesAsync()`). Esto reduce drásticamente las transacciones de red a una sola operación masiva.

---

## Parte 2: Implementación Práctica en C#

### Desafío 1: Consumo de Endpoints y Deserialización

A continuación, se presenta la implementación del método asíncrono y seguro para consumir el listado de alumnos:

```csharp
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

public class ConectividadAcademica
{
    private readonly HttpClient _httpClient;

    public ConectividadAcademica(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<Alumno?> ObtenerAlumnoAsync()
    {
        string url = "[https://api.usac.edu/v1/alumnos](https://api.usac.edu/v1/alumnos)";

        try
        {
            // Realizar la petición GET segura
            HttpResponseMessage response = await _httpClient.GetAsync(url);
            
            // Validar el código de estado HTTP (Lanza HttpRequestException si no es 2xx)
            response.EnsureSuccessStatusCode();

            // Leer el payload de la respuesta de forma asíncrona
            string jsonResponse = await response.Content.ReadAsStringAsync();

            // Configurar las opciones para habilitar insensibilidad a mayúsculas y minúsculas
            var options = new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            };

            // Deserializar el payload JSON al objeto Alumno
            Alumno? alumno = JsonSerializer.Deserialize<Alumno>(jsonResponse, options);
            return alumno;
        }
        catch (HttpRequestException httpEx)
        {
            // Manejo específico de errores de comunicación HTTP
            Console.WriteLine($"Error de comunicación con la API de la USAC: {httpEx.Message}");
            throw;
        }
        catch (JsonException jsonEx)
        {
            // Manejo específico de errores de parseo o formato JSON inválido
            Console.WriteLine($"Error en el formato de deserialización: {jsonEx.Message}");
            throw;
        }
        catch (Exception ex)
        {
            // Manejo de imprevistos generales
            Console.WriteLine($"Ocurrió un error inesperado: {ex.Message}");
            throw;
        }
    }
}

// Clase de modelo para representar la entidad Alumno
public class Alumno
{
    public int Id { get; set; }
    public string Carne { get; set; } = string.Empty;
    public string Nombre { get; set; } = string.Empty;
    public string Apellido { get; set; } = string.Empty;
}




### Desafío 2: Endpoint para Carga Masiva CSV

A continuación se presenta el desarrollo de un endpoint en C# (`[HttpPost]`) para un controlador web que recibe un archivo masivo mediante `IFormFile`. La lógica interna está diseñada bajo estrictos estándares de rendimiento para el procesamiento asíncrono y la inserción eficiente por lotes:

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

namespace SistemaControlAcademico.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class CargaMasivaController : ControllerBase
    {
        private readonly MyDbContext _context; // Contexto de persistencia (Entity Framework Core)

        public CargaMasivaController(MyDbContext context)
        {
            _context = context;
        }

        /// <summary>
        /// Endpoint para la carga masiva de alumnos a partir de un archivo CSV.
        /// </summary>
        /// <param name="archivoCsv">Archivo físico enviado mediante form-data.</param>
        [HttpPost("cargar-alumnos-csv")]
        public async Task<IActionResult> CargarAlumnosCsv(IFormFile archivoCsv)
        {
            // Validación preliminar del archivo cargado
            if (archivoCsv == null || archivoCsv.Length == 0)
            {
                return BadRequest("El archivo proporcionado está vacío o es inválido.");
            }

            // Lista intermedia en memoria para aplicar la estrategia de Batching (Lotes)
            var listaAlumnosBatch = new List<AlumnoEntidad>();

            try
            {
                // Flujo asíncrono: Se abre el stream del archivo evitando cargar todo el peso en la RAM
                using (var stream = archivoCsv.OpenReadStream())
                using (var reader = new StreamReader(stream))
                {
                    string? linea;
                    bool esPrimeraLinea = true;

                    // Procesamiento línea por línea para mitigar la saturación de memoria en el servidor
                    while ((linea = await reader.ReadLineAsync()) != null)
                    {
                        // Saltar de forma segura la línea de cabecera del CSV
                        if (esPrimeraLinea)
                        {
                            esPrimeraLinea = false;
                            continue;
                        }

                        // Ignorar líneas en blanco accidentales
                        if (string.IsNullOrWhiteSpace(linea)) continue;

                        // Parseo de los campos delimitados por comas
                        string[] columnas = linea.Split(',');

                        if (columnas.Length >= 3)
                        {
                            var nuevoAlumno = new AlumnoEntidad
                            {
                                Carne = columnas[0].Trim(),
                                Nombre = columnas[1].Trim(),
                                Apellido = columnas[2].Trim()
                            };

                            // Acumulación en la lista intermedia
                            listaAlumnosBatch.Add(nuevoAlumno);
                        }
                    }
                }

                // Si se encontraron registros válidos, se procesa la inserción masiva
                if (listaAlumnosBatch.Count > 0)
                {
                    // Registro del bloque completo en el ChangeTracker de EF
                    await _context.Alumnos.AddRangeAsync(listaAlumnosBatch);

                    // Única transacción/llamada a la base de datos al concluir la lectura
                    await _context.SaveChangesAsync();
                }

                return Ok(new { Mensaje = $"Carga masiva finalizada con éxito. {listaAlumnosBatch.Count} registros insertados." });
            }
            catch (Exception ex)
            {
                // Manejo de excepciones ante fallos de servidor o de base de datos
                return StatusCode(StatusCodes.Status500InternalServerError, 
                    $"Error interno del servidor durante el procesamiento: {ex.Message}");
            }
        }
    }

    // --- Modelos de persistencia requeridos para el ejemplo ---

    public class AlumnoEntidad
    {
        public int Id { get; set; } // Llave primaria autoincrementable
        public string Carne { get; set; } = string.Empty;
        public string Nombre { get; set; } = string.Empty;
        public string Apellido { get; set; } = string.Empty;
    }

    public class MyDbContext : DbContext
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options) { }
        public DbSet<AlumnoEntidad> Alumnos { get; set; } = null!;
    }
}


## Parte 3: Referencias Bibliográficas (Requisito Formal)

* Facultad de Ingeniería, USAC. (2026). Sesión 20: Integración de Datos. Consumo de APIs Externas y Carga Masiva (CSV/XML). Laboratorio del curso Introducción a la Programación y Computación 2. Guatemala.[cite: 1]