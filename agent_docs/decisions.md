# Decisions (ADR)

## 001 — xUnit + FluentAssertions как тестовый стек

**Решение**: xUnit + FluentAssertions для всех генерируемых проектов.
**Почему**: xUnit — стандарт де-факто для .NET, FluentAssertions даёт читаемые assertion'ы.
**Отвергнуто**: NUnit (менее популярен в новых проектах), MSTest (слабее расширяемость).
