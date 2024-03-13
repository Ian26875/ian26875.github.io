---
title: [SOLID原則:單一職責原則(Single Responsibility Principle)]
date: 2023-04-23 13:23:48

tags:
  - 2023
  - OOP
  - SOLID

categories:
  - [OOP,SOLID]

keywords: 

  - OOP
  - SOLID

---

# 單一職責原則(Single Responsibility Principle)


> 一個模組應該只對唯一的一個角色負責


一個類別或模塊只應該負責一個功能或職責。這個原則有助於降低代碼的複雜度，使代碼更容易維護和擴展。這避免了一個類別出現了上帝類別（God Class)。

舉例來說，這是一個員工類別。


```csharp

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Salary { get; set; }
    public decimal CommissionRate { get; set; }
    public decimal BonusRate { get; set; }

    public void CalculateSalary()
    {
        // 計算薪水的邏輯
        Salary = // ...
    }

    public void CalculateCommission()
    {
        // 計算佣金的邏輯
    }

    public void CalculateBonus()
    {
        // 計算獎金的邏輯
    }

    public void Sell()
    {
        // 銷售的邏輯
    }

    public void DevelopSoftware()
    {
        // 軟體開發的邏輯
    }

}

```



這類別包含很多方法。包括表示員工的數據、計算薪水、銷售、軟體開發和需求修改等功能。這導致代碼複雜度提高，使得這個類更難理解、修改和擴展。為了解決這個問題，可以使用單一職責原則。


```csharp

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Salary { get; set; }
}

public class FinanceEmployee : Employee
{
    public void ManageAccounts()
    {
        // 財務員工管理帳戶的邏輯
    }
}

public class ITEmployee : Employee
{
    public void DevelopSoftware()
    {
        // IT員工開發軟體的邏輯
    }
}

public class SalesEmployee : Employee
{
    public decimal CommissionRate { get; set; }

    public void Sell()
    {
        // 銷售人員銷售的邏輯
    }
}

```


將 Employee 類分成 FinanceEmployee、ITEmployee 和 SalesEmployee 三個類。FinanceEmployee 類負責財務管理的功能，ITEmployee 類負責開發軟體的功能，SalesEmployee 類負責銷售的功能。每個類都只負責一個職責，從而使代碼更易於理解、修改和擴展。






