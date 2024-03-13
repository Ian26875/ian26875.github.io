---
title: 設計抽象介面 
date: 2023-04-16 21:22:48
tags:
  - 2023
  - Interface

categories:
  - [OOP]

keywords: 
  - 介面設計

---
# 抽象介面設計 - 抽象洩漏

在軟體開發上使用抽象介面是很常用的手段，但是介面設計往往沒有很好的檢視。導致後續使用介面遇到許多問題，甚至質疑使用介面。往往不是介面的錯，是設計介面品質出錯。導致抽象並沒有很好的發揮該有的隱蔽細節、簡化問題的功能。


---

## 什麼是抽象洩漏?

「抽象泄漏」是軟體開發時，本應隱藏實現細節的抽象化不可避免地暴露出底層細節與局限性。抽象泄露是棘手的問題，因為抽象化本來目的就是向用戶隱藏不必要公開的細節。(From - [維基百科](https://https://zh.wikipedia.org/zh-tw/%E6%8A%BD%E8%B1%A1%E6%B3%84%E6%BC%8F))

以下面的介面為例:

```csharp!
public interface IStudentDbRepository
{
    
    Task<StudentModel> GetByIdAsync(string studentId);
    
    Task<StudentModel> GetByNameAsync(string studentName);
    
    Task SaveAsync(StudentModel student);
    
    Task ModifyAsync(StudentModel student);
    
    Task RemoveAsync(StudentModel student);
}

```

---

## 要如何避免抽象洩漏??

在```IStudentDbRepository.cs```之中，介面名稱已經透露出實作內容是依賴資料庫。
這是一個很不好的介面設計。當實作方式從資料庫存取改為使用Web API方式，這介面是不是會出現問題 ?


比較好一點的介面設計應該是隱蔽一些實作的細節。在Repository的介面中，我們應該需要隱蔽的是資料從哪邊來，讓調用方可以直接從介面之中取得自己所需的資料，不必知道資料從Database或是Web API。這是實作 Repository 介面才需要決定的，並非其他調用方需要知道。並不是為了抽介面而建立介面。

```csharp!
public interface IStudentRepository
{
    
    Task<StudentModel> GetByIdAsync(string studentId);
    
    Task<StudentModel> GetByNameAsync(string studentName);
    
    Task SaveAsync(StudentModel student);
    
    Task ModifyAsync(StudentModel student);
    
    Task RemoveAsync(StudentModel student);
}

```