# Wheelzy

__1. Define the database tables with its corresponding data types and relationships for the
following case:
A customer wants to sell his car so we need the car information (car year, make, model, and submodel), the location of the car (zip code) and the list of buyers that are willing to buy the car in that zip code. Each buyer will have a quote (amount) and only one buyer will be marked as the current one (not necessarily the one with the highest quote). We also want to track the progress of the case using different statuses (Pending Acceptance, Accepted, Picked Up, etc.) and considering that we care about the current status and the status history (previous statuses indicate when it happened, who changed it, etc.). The status “Picked Up” has a mandatory status date but the rest of the statuses don’t.
Write a SQL query to show the car information, current buyer name with its quote and current status name with its status date. Do the same thing using Entity Framework.
Make sure your queries don’t have any unnecessary data.__

**1.1 Solution:**

Construccion de la base de datos

```sql
Create Database Wheelzy
go

use Wheelzy
go

Create table Location(
idLocation int not null IDENTITY(1,1) PRIMARY KEY,
zipCode varchar(10) not null
);

Create Table Car(
idCar int not null IDENTITY(1,1) PRIMARY KEY,
carYear int not null,
make varchar(40) not null,
model varchar (40) not null,
submodel varchar(40) not null,
idLocation int not null,
CONSTRAINT pk_zip_Code FOREIGN KEY(idLocation)  REFERENCES Location(idLocation)
);

Create Table Buyer(
idBuyer int not null primary key identity(1,1),
name varchar(40) not null
);

Create Table Quotes(
idQuotes int not null IDENTITY(1,1) PRIMARY KEY,
idCar int not null,
idBuyer int not null,
amount float not null,
isCurrentBuyer bit not null,
CONSTRAINT pk_pkIdCar FOREIGN KEY(idCar)  REFERENCES Car(idCar),
CONSTRAINT pk_pkIdBuyer FOREIGN KEY(idBuyer)  REFERENCES Buyer(idBuyer)
);

CREATE TABLE Status(
idStatus int not null PRIMARY KEY IDENTITY(1,1),
name varchar(30) not null
);

CREATE TABLE StatusHistory(
idStatusHistory int not null PRIMARY KEY IDENTITY(1,1),
idCar int not null,
idStatus int not null,
statusDate datetime,
statusChangeby varchar(40),
CONSTRAINT pk_idCar_CaseProgress FOREIGN KEY(idCar) REFERENCES Car(idCar),
CONSTRAINT pk_idStatus_CaseProgress FOREIGN KEY(idStatus) REFERENCES Status(idStatus)
);
```
**Consulta sql**

El parámetro @idCard es para indicar la información del carro que deseamos ver.

```sql
SELECT c.carYear, c.make, c.model, c.subModel, l.zipCode, b.name buyerName, q.amount buyerQuote, s.name currentStatus, sh.statusDate
FROM Car c
JOIN Location l on c.idLocation = l.idLocation
JOIN Quotes q on c.idCar = q.idCar
JOIN Buyer b on q.idBuyer = b.idBuyer
JOIN StatusHistory sh on c.idCar = sh.idCar
JOIN Status s on cp.idStatus = s.idStatus
where q.isCurrentBuyer = 1 and c.idCar = @idCard
```

**Consulta EF**

Realizamos el mapeo de las entidades
```csharp
[Table("Location")]
public class Location
{
  [Key]
  public int idLocation { get; set; }
  public string zipCode { get; set; }
}
```
```csharp
[Table("Car")]
public class Car
{
  [Key]
  public int idCar { get; set; }
  public int carYear { get; set; }
  public string make { get; set; }
  public string model { get; set; }
  public string submodel { get; set; }
  public int idLocation { get; set; }
}
```
```csharp
[Table("Buyer")]
public class Buyer
{
  [Key]
  public int idBuyer { get; set; }
  public string name { get; set; }
}
```
```csharp
[Table("Quotes")]
public class Quotes
{
  [Key]
  public int idQuotes { get; set; }
  public int idCar { get; set; }
  public int idBuyer { get; set; }
  public float amount { get; set; }
  public bool isCurrentBuyer { get; set; }
}
```
```csharp
[Table("Status")]
public class Status
{
  [Key]
  public int idStatus { get; set; }
  public string name { get; set; }
}
```
```csharp
[Table("StatusHistory")]
public class StatusHistory
{
  [Key]
  public int idStatusHistory { get; set; }
  public int idCar { get; set; }
  public int idStatus { get; set; }
  public datetime statusDate { get; set; }
  public string statusChangeby { get; set; }
}
```
Procedemos a crear el contexto, para esto simulamos ya haber instalado Entity Framework
```csharp
public partial class ModelContext:DbContext
{
   public virtual DbSet<Location> Locations { get; set; }
   public virtual DbSet<Car> Cars { get; set; }
   public virtual DbSet<Buyer> Buyers { get; set; }
   public virtual DbSet<Quotes> Quotes { get; set; }
   public virtual DbSet<Status> Status { get; set; }
   public virtual DbSet<StatusHistory> StatusHistorys { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("adena_de_conexion");
    }
}
```

Relizado el mapeo de las entidades y la construccion del context, pocedemos a crear un metodo para realizar la consulta
```csharp
public class CarSalesInformation
{
    public void Get(int idCard)
    {
        using (ModelContext modeldb = new ModelContext())
        {
            var query = (from c in modeldb.Cars
                         join l in modeldb.Locations on c.idLocaion equals l.idLocation
                         join q in modeldb.Quotes on c.idCar equals q.idCar
                         join b in modeldb.Buyers on q.idBuyer equals b.idBuyer
                         join sh in modeldb.StatusHistorys on c.idCar equals sh.idcar
                         join s in modeldb.Status on sh.idStatus equals s.idStatus
                         where q.isCurrentBuyer == true && c.idCar == idCard
                         select new { c.carYear, c.make, c.model, c.subModel, l.zipCode, b.name buyerName, q.amount buyerQuote, s.name currentStatus, sh.statusDate }
                        ).FirstOrDefault();
        }
    }
}
```
__2. What would you do if you had data that doesn’t change often but it’s used pretty much all the time?__

Implementaria :
  - La optimatización de las consultas existentes para que no obtengan información innecesaria.
  - La creación de indices a las tablas involucradas para mejorar el rendimiento en la respuesta de la consulta.
  - La creación de una base de datos no relacional como __Mongo DB__ con el fin de tener la información mas detallada en una collecion.
  - Uso del cache de la información mediante Redis.


 __3. Analyze the following method and make changes to make it better. Explain your
changes.__
```csharp
public void UpdateCustomersBalanceByInvoices(List<Invoice> invoices)
{
  foreach (var invoice in invoices)
  {
    var customer = dbContext.Customers.SingleOrDefault(invoice.CustomerId.Value);
    customer.Balance -= invoice.Total;
    dbContext.SaveChanges();
  }
}
```
**3.1 Solution:**
```csharp
public void UpdateCustomersBalanceByInvoices(List<Invoice> invoices)
{
  foreach (var invoice in invoices)
  {
    var customerId = invoice.CustomerId;
    if(customerId != null)
    {
      var customer = dbContext.Customers.SingleOrDefault(c=> c.CustomerId == customerId);
      if(customer != null)
        customer.Balance -= invoice.Total;
    }
  }
    dbContext.SaveChanges();
}
```
__4. Implement the following method using Entity Framework, making sure your query is
efficient in all the cases (when all the parameters are set, when some of them are or
when none of them are). If a “filter” is not set it means that it will not apply any filtering
over that field (no ids provided for customer ids it means we don’t want to filter by
customer).__

**Solution**
Para dar solución, supongamos que las entidades, el contexto ya estan creadas y  el **OrderDTO** contiene las siguientes propiedades
```csharp
public clas OrderDTO
{
  public int OrderId { get; set; }
  public string CustomerName { get; set; }
  public DateTime OrderDate { get; set; }
}
```

```csharp
public async Task<OrderDTO> GetOrders(DateTime dateFrom, DateTime dateTo, List<int> customerIds, List<int> statusIds, bool? isActive)
{
  // your implementation
  try
  {
    OrderDTO orderDTO = new OrderDTO();
    var query = modelContext.Orders.AsQueryable();
    if(dateFrom != DateTime.MinValue)
    {
      query = query.Where(o=> o.OrderDate >= dateFrom);
    }
  
    if(dateTo != DateTime.MinValue)
    {
      query = query.Where(o=> o.OrderDate <= dateTo);
    }
  
    if(customerIds != null && customerIds.Count > 0)
    {
      query = query.Where(o=> customerIds.Contains(o.CustomerId));
    }
  
    if(statusIds != null && statusIds.Count > 0)
    {
      query = query.Where(o=> statusIds.Contains(o.statusId));
    }
  
    if(isActive != null)
    {
      query = query.Where(o=> o.IsActive == isActive);
    }
    orderDTO = query.Select(o=> new OrderDTO(){
      OrderId = o.OrderId,
      CustomerName = o.CustomerName,
      OrderDate = o.OrderDate
    }).FirstOrDefault();
  }
  catch(Exception ex)
  {
    throw;
  }
  return orderDTO;
}
```

__5. Bill, from the QA Department, assigned you a high priority task indicating there’s a bug when someone changes the status from “Accepted” to “Picked Up”.
Define how you would proceed, step by step, until you create the Pull Request.__

Procederia de la siguiente forma:
- Crear una rama del repositorio de codigo fuente para darle solución al bug.
- Analizar la parte del codigo que tiene relación con la funcionalidad involucrada.
- Revisaria el log de la aplicación si llega a contar con uno implementado, con el fin de obtener mas detalle.
- Intentar reproducir el escenario del error.
- Realizar una depuración del codigo con el fin de ver el flujo de ejecución y detectar la posible causa.
- Realizar la solución del bug.
- Realizar pruebas
- Documentar la solución del bug
- Crear el Pull Request con la rama que contiene la solución
