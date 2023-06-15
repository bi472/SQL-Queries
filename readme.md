
## Задача 2
Вывести общую сумму продаж с разбивкой по годам и месяцам, за все время работы компании
```sql
SELECT 
	YEAR([ModifiedDate]) as SalesYear,
    Month([ModifiedDate]) as SalesMonth,
	SUM(LineTotal) as TotalSales
FROM 
	[AdventureWorks2017].[Sales].[SalesOrderDetail]
GROUP BY
	YEAR([ModifiedDate]),
	Month([ModifiedDate])
ORDER BY
	YEAR([ModifiedDate]),
	Month([ModifiedDate])
```

## Задача 3
Выбрать 10 самых приоритетных городов для следующего магазина
Столбцы: Город | Приоритет
Приоритет определяется как количество покупателей в городе
В городе не должно быть магазина
```SQL
SELECT TOP 10
    PA.City AS City,
    COUNT(*) AS Priority
FROM
    Sales.Customer C
JOIN
	Sales.SalesOrderHeader SOH ON C.CustomerID = SOH.CustomerID
JOIN
    Person.Address PA ON SOH.ShipToAddressID = PA.AddressID
WHERE PA.City NOT IN (
	SELECT City
	FROM Sales.vStoreWithAddresses
)
GROUP BY
    PA.City
ORDER BY
    COUNT(*) DESC;
```
## Задача 4
Выбрать покупателей, купивших больше 15 единиц одного и того же продукта за все время работы компании.
Столбцы: Фамилия покупателя | Имя покупателя | Название продукта | Количество купленных экземпляров (за все время) 
Упорядочить по количеству купленных экземпляров по убыванию, затем по полному имени покупателя по возрастанию
```SQL
SELECT
    P.LastName AS 'Last Name',
    P.FirstName AS 'First Name',
    PR.Name AS 'Product Name',
    SUM(SOD.OrderQty) AS 'Total Quantity Purchased'
FROM
    Sales.Customer AS C
JOIN
    Person.Person AS P ON C.PersonID = P.BusinessEntityID
JOIN
    Sales.SalesOrderHeader AS SOH ON C.CustomerID = SOH.CustomerID
JOIN
    Sales.SalesOrderDetail AS SOD ON SOH.SalesOrderID = SOD.SalesOrderID
JOIN
    Production.Product AS PR ON SOD.ProductID = PR.ProductID
GROUP BY
    P.LastName,
    P.FirstName,
    PR.Name
HAVING
    SUM(SOD.OrderQty) > 15
ORDER BY
    SUM(SOD.OrderQty) DESC,
    P.FirstName ASC,
    P.LastName ASC;
```

## Задача 5
Вывести содержимое первого заказа каждого клиента
Столбцы: Дата заказа | Фамилия покупателя | Имя покупателя | Содержимое заказа
Упорядочить по дате заказа от новых к старым
В ячейку содержимого заказа нужно объединить все элементы заказа покупателя в следующем формате:
<Имя товара> Количество: <количество в заказе> шт.
<Имя товара> Количество: <количество в заказе> шт.
<Имя товара> Количество: <количество в заказе> шт.
...
```SQL
SELECT
    SOH.OrderDate AS 'Order Date',
    P.LastName AS 'Last Name',
    P.FirstName AS 'First Name',
    STRING_AGG(CONCAT(PR.Name, ' Quantity: ', SOD.OrderQty, ' pcs.'), ', ') AS 'Order Contents'
FROM
    Sales.Customer AS C
JOIN
    Person.Person AS P ON C.PersonID = P.BusinessEntityID
JOIN
    Sales.SalesOrderHeader AS SOH ON C.CustomerID = SOH.CustomerID
JOIN
    Sales.SalesOrderDetail AS SOD ON SOH.SalesOrderID = SOD.SalesOrderID
JOIN
    Production.Product AS PR ON SOD.ProductID = PR.ProductID
WHERE
    SOH.SalesOrderID IN (
        SELECT MIN(SalesOrderID)
        FROM Sales.SalesOrderHeader
        GROUP BY CustomerID
    )
GROUP BY
    SOH.OrderDate,
    P.LastName,
    P.FirstName
ORDER BY
    SOH.OrderDate DESC;
```

## Задача 6
Вывести список сотрудников, непосредственный руководитель которых младше сотрудника и меньше работает в компании
Столбцы: Имя руководителя | Дата приема руководителя на работу | Дата рождения руководителя |
	| Имя сотрудника | Дата приема сотрудника на работу | Дата рождения сотрудника
Поле имя выводить в формате 'Фамилия И.О.'
Упорядочить по иерархии от директора вниз к рядовым сотрудникам
Внутри одного уровня иерархии упорядочить по фамилии руководителя, затем по фамилии сотрудника

```SQL
SELECT
    CONCAT(P_MGR.LastName, ' ', LEFT(P_MGR.FirstName, 1), '.') AS 'Manager Name',
    MGR.HireDate AS 'Manager Hire Date',
    MGR.BirthDate AS 'Manager Birth Date',
    CONCAT(P_EMP.LastName, ' ', LEFT(P_EMP.FirstName, 1), '.') AS 'Employee Name',
    EMP.HireDate AS 'Employee Hire Date',
    EMP.BirthDate AS 'Employee Birth Date'
FROM
    HumanResources.Employee AS EMP
JOIN
    Person.Person AS P_EMP ON EMP.BusinessEntityID = P_EMP.BusinessEntityID
JOIN
    HumanResources.Employee AS MGR ON EMP.OrganizationNode.GetAncestor(1) = MGR.OrganizationNode
JOIN
    Person.Person AS P_MGR ON MGR.BusinessEntityID = P_MGR.BusinessEntityID
WHERE
    MGR.BirthDate > EMP.BirthDate
    AND MGR.HireDate > EMP.HireDate
	AND MGR.OrganizationLevel = EMP.OrganizationLevel - 1
ORDER BY
    EMP.OrganizationNode ASC,
    P_MGR.LastName ASC,
    P_EMP.LastName ASC;
```

## Задача 7
Написать хранимую процедуру, с тремя параметрами и результирующим набором данных 
входные параметры - две даты, с и по 
выходной параметр - количество найденных записей 
Результирующий набор содержит записи всех холостых мужчин-сотрудников, родившихся в диапазон указанных в параметре дат

```SQL
CREATE PROCEDURE GetUnmarriedMaleEmployees
    @StartDate DATE,
    @EndDate DATE,
    @RecordCount INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT P.FirstName, P.LastName, E.BirthDate
    INTO #Results
    FROM HumanResources.Employee AS E
    JOIN Person.Person AS P ON E.BusinessEntityID = P.BusinessEntityID
    WHERE E.Gender = 'M' -- Мужчины
        AND E.MaritalStatus = 'S' -- Холостые
        AND E.BirthDate BETWEEN @StartDate AND @EndDate; -- Родившиеся в указанном диапазоне дат

    SELECT @RecordCount = COUNT(*)
    FROM #Results;

    SELECT FirstName, LastName, BirthDate
    FROM #Results
    ORDER BY LastName, FirstName;

    DROP TABLE IF EXISTS #Results;
END
```