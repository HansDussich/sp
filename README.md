# sp

📌 1. Configuración Inicial

1.1. Crear la Base de Datos en SQL Server

CREATE DATABASE EmpleadosDB;
GO

USE EmpleadosDB;
GO

CREATE TABLE Empleados (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    Nombre NVARCHAR(100) NOT NULL,
    Correo NVARCHAR(100) UNIQUE NOT NULL,
    FechaNacimiento DATE NOT NULL
);

1.2. Stored Procedures (SP) para CRUD

-- Obtener todos los empleados
CREATE PROCEDURE sp_GetEmpleados
AS
BEGIN
    SELECT * FROM Empleados;
END;
GO

-- Obtener empleado por ID
CREATE PROCEDURE sp_GetEmpleadoById @Id INT
AS
BEGIN
    SELECT * FROM Empleados WHERE Id = @Id;
END;
GO

-- Insertar empleado
CREATE PROCEDURE sp_InsertEmpleado @Nombre NVARCHAR(100), @Correo NVARCHAR(100), @FechaNacimiento DATE
AS
BEGIN
    INSERT INTO Empleados (Nombre, Correo, FechaNacimiento) VALUES (@Nombre, @Correo, @FechaNacimiento);
    SELECT SCOPE_IDENTITY() AS Id;
END;
GO

-- Actualizar empleado
CREATE PROCEDURE sp_UpdateEmpleado @Id INT, @Nombre NVARCHAR(100), @Correo NVARCHAR(100), @FechaNacimiento DATE
AS
BEGIN
    UPDATE Empleados SET Nombre = @Nombre, Correo = @Correo, FechaNacimiento = @FechaNacimiento WHERE Id = @Id;
END;
GO

-- Eliminar empleado
CREATE PROCEDURE sp_DeleteEmpleado @Id INT
AS
BEGIN
    DELETE FROM Empleados WHERE Id = @Id;
END;
GO

📌 2. Creación del Proyecto en .NET 8

dotnet new webapi -n MicroservicioEmpleados
cd MicroservicioEmpleados

Instalar paquetes necesarios:

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design

📌 3. Configurar la Conexión a SQL Server

Editar appsettings.json:

"ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EmpleadosDB;User Id=sa;Password=TuContraseña;TrustServerCertificate=True;"
}

📌 4. Crear los Modelos en C#

4.1. Modelo Empleados.cs

using System.ComponentModel.DataAnnotations;

namespace MicroservicioEmpleados.Models
{
    public class Empleados
    {
        [Key]
        public int Id { get; set; }
        public string Nombre { get; set; } = string.Empty;
        public string Correo { get; set; } = string.Empty;
        public DateTime FechaNacimiento { get; set; }
    }
}

4.2. EmpleadosDbContext.cs

using Microsoft.EntityFrameworkCore;

namespace MicroservicioEmpleados.Models
{
    public class EmpleadosDbContext : DbContext
    {
        public EmpleadosDbContext(DbContextOptions<EmpleadosDbContext> options) : base(options) { }
        public DbSet<Empleados> Empleados { get; set; }
    }
}

📌 5. Agregar el Contexto en Program.cs

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<EmpleadosDbContext>(options => options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();

📌 6. Generar el Controlador con Scaffolding

dotnet aspnet-codegenerator controller -name EmpleadosController -async -api -m Empleados -dc EmpleadosDbContext -outDir Controllers

6.1. EmpleadosController.cs Usando SP

using Microsoft.AspNetCore.Mvc;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using MicroservicioEmpleados.Models;
using System.Data;

namespace MicroservicioEmpleados.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class EmpleadosController : ControllerBase
    {
        private readonly EmpleadosDbContext _context;
        private readonly string _connectionString;

        public EmpleadosController(EmpleadosDbContext context, IConfiguration config)
        {
            _context = context;
            _connectionString = config.GetConnectionString("DefaultConnection");
        }

        [HttpGet]
        public async Task<IActionResult> GetEmpleados()
        {
            using var conn = new SqlConnection(_connectionString);
            using var cmd = new SqlCommand("sp_GetEmpleados", conn) { CommandType = CommandType.StoredProcedure };
            conn.Open();
            var empleados = new List<Empleados>();
            using var reader = await cmd.ExecuteReaderAsync();
            while (reader.Read())
            {
                empleados.Add(new Empleados
                {
                    Id = reader.GetInt32(0),
                    Nombre = reader.GetString(1),
                    Correo = reader.GetString(2),
                    FechaNacimiento = reader.GetDateTime(3)
                });
            }
            return Ok(empleados);
        }
    }
}

📌 7. Ejecutar y Probar la API

dotnet run

Visitar en navegador:

http://localhost:5000/swagger

O probar con curl:

curl -X GET "http://localhost:5000/api/Empleados"

📌 8. Desplegar con Docker

Crear Dockerfile:

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .
CMD ["dotnet", "MicroservicioEmpleados.dll"]

Construir y ejecutar:

docker build -t microservicio-empleados .
docker run -p 5000:80 microservicio-empleados

