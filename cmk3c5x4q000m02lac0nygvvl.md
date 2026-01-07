---
title: "The Realization: Why I Needed TDD"
seoTitle: "TDD"
seoDescription: "TDD"
datePublished: Wed Jan 07 2026 01:24:46 GMT+0000 (Coordinated Universal Time)
cuid: cmk3c5x4q000m02lac0nygvvl
slug: the-realization-why-i-needed-tdd
tags: tdd

---

## My First Encounter with Test Code

---

When I first joined my previous company, I found that despite being a solution provider, there wasn't a properly established core solution. We had one product delivered to a client, but the codebase—likely due to a rushed timeline—lacked consistent conventions. I immediately felt that this project desperately needed verification for its tangled logic and various technical debts.

As we began building a new version of the solution, I proposed the introduction of test codes to my colleagues. Although they had little experience with automated testing, and I hadn't written extensive test code in a production environment myself, we all agreed on its necessity and decided to move forward.

## Feeling the Necessity of TDD

---

The journey wasn't exactly smooth, but following the footsteps of many senior developers online, I gradually integrated test codes into the project. I added tests to existing business logic and established rules for different testing scopes. At the time, I thought I was writing well-organized, rule-compliant test code.

However, at some point, I felt that something was missing. Because I was writing tests *after* the business logic was already finished, the tests couldn't influence the design itself. In some cases, the tests ended up merely validating poorly designed or flawed logic, making them feel less than half-effective.

That was when **TDD (Test-Driven Development)** caught my eye. TDD seemed like the perfect approach to improve both design capabilities and requirement verification. Although it felt like a daunting concept at first, I began learning it through formal training. This post is a record of those trials and errors.

## Feedback Summary (Before the 3rd Session)

---

1. Use `hasSize()` when validating the size of a Collection.
    
2. Parameters in `@ParameterizedTest` support automatic type conversion.
    
3. Replace Magic Numbers and Magic Strings with constants.
    
4. Do not abbreviate variable names.
    
5. Minimize the use of getters and setters.
    
6. Consider the specific nuance of variable names (e.g., `check` vs. `validate` vs. `verify`).
    
7. If something is difficult to test (like random values), push the responsibility to a higher-level object.
    
8. Instead of using a getter, send a message to the object (Ask, don't tell).
    
9. Keep indentation levels to a maximum of one.
    
10. Limit classes to a maximum of 2-3 instance variables.
    
11. Wrap primitive values in classes (Value Objects).
    
12. Utilize First-Class Collections.
    
13. Follow the "Object Calisthenics" principles.
    

## Object Calisthenics

---

Rather than viewing these as rigid laws, I treat them as guidelines that increase the probability of producing high-quality code.

1. Only one level of indentation per method.
    
2. Don't use the `else` keyword.
    
3. Wrap all primitives and strings.
    
4. Use only one dot per line (e.g., `object.method()`, avoiding deep chaining).
    
5. Don't abbreviate.
    
6. Keep all entities small.
    
7. No classes with more than three instance variables.
    
8. Use First-Class Collections.
    
9. No getters, setters, or properties (where possible).