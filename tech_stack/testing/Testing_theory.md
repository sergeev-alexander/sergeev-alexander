**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Testing Theory

## 7 принципов тестирования

> - **Exhaustive testing is impossible** (Исчерпывающее тестирование невозможно).<br>Полный перебор всех возможных
    сценариев невозможен из-за бесконечного числа комбинаций данных и условий.
>
> 
> - **Testing shows the presence of defects, not their absence** (Тестирование демонстрирует наличие дефектов, а не их
    отсутствие)<br>Тестирование может выявить ошибки, но не гарантирует их полного отсутствия в продукте.
>
> 
> - **Absence-of-errors fallacy** (Заблуждение об отсутствии ошибок)<br>Даже после исправления многих дефектов нельзя
    утверждать, что продукт идеален и готов к использованию, так как требования пользователей могут различаться.
>
> 
> - **Early testing saves time and money** (Раннее тестирование сохраняет время и деньги)<br>Чем раньше обнаружен
    дефект, тем дешевле и проще его исправить, предотвращая накопление проблем на поздних этапах.
>
> 
> - **Defect clustering** (Принцип скопления или кластеризация дефектов)<br>Большинство дефектов обычно сосредоточено в
    небольшом количестве модулей, что требует к ним особого внимания при тестировании.
>
> 
> - **Testing is context dependent** (Тестирование зависит от контекста)<br>Подход к тестированию зависит от типа
    продукта, его целей, команды, сроков и доступных инструментов, поэтому стратегия всегда адаптируется.
>
> 
> - **Pesticide paradox** (Парадокс пестицида)<br>Повторение одних и тех же тестов со временем становится неэффективным
    для поиска новых дефектов, поэтому тестовые сценарии необходимо регулярно обновлять.

---

## Классификация тестирования

### 1. По уровню (масштабу) тестирования (By Testing Level): `Модульное` → `Интеграционное` → `Системное` → `Приемочное`.

### 2. По типу (объекту) тестирования (By Test Type): `Функциональное` / `Нефункциональное` / `Основанное на изменениях`.

### 3. По уровню доступа к коду / системе (By System Knowledge): `Black` / `White` / `Grey Box`.

### 4. По исполнению кода (By Test Execution): `Статическое` / `Динамическое`.

### 5. По степени автоматизации (By Automation Level): `Ручное` / `Автоматизированное`.

---

## 1. УРОВНИ ТЕСТИРОВАНИЯ (Testing Levels)

- `Unit Testing` (Модульное / Компонентное тестирование): Тестирование отдельных изолированных модулей или компонентов (
  обычно методов или классов) программы.
- `Integration Testing` (Интеграционное тестирование): Тестирование взаимодействия между несколькими модулями,
  компонентами или системами.
- `System Testing` (Системное тестирование): Тестирование полностью интегрированной системы на соответствие всем
  требованиям.
- `Acceptance Testing` (Приемочное тестирование): Финальное тестирование, проводимое с целью принятия решения о
  готовности системы к выпуску.

---

## 2. ТИПЫ ТЕСТИРОВАНИЯ (Testing Types)

> Классификация по цели и объекту проверки. Один и тот же тест может относиться к разным типам в зависимости от аспекта,
> который он проверяет.

### 2.1 Functional Testing (Функциональное тестирование)

> Проверка того, **ЧТО** делает система — соответствие бизнес-требованиям и спецификациям.

- [`Functional Testing Types` (Типы функционального тестирования)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Functional_testing_types.md "Functional_testing_types.md")
  <br>
  Проверка соответствия функциональности системы заявленным требованиям.

  Примеры: Smoke Testing, Sanity Testing, End-to-End (E2E), User Acceptance Testing (UAT), Interoperability
  Testing.

### 2.2 Non-functional Testing (Нефункциональное тестирование)

> Проверка того, **КАК** работает система — качественные характеристики и атрибуты.

- [`Non-functional Testing Types` (Типы нефункционального тестирования)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Non-functional_testing_types.md "Non-functional_testing_types.md")
  <br>
  Проверка качественных характеристик системы (производительность, безопасность, удобство использования и т.д.).

  Примеры: Performance Testing (Load, Stress, Spike), Security Testing, Usability, Accessibility, Compatibility, Reliability, Maintainability.

### 2.3 Change-related Testing (Тестирование изменений)

> Проверка корректности модификаций и отсутствия регрессии после внесения изменений.

- [`Change-related Testing Types` (Типы тестирования, основанные на изменениях)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Change-Related_testing_types.md "Change-Related_testing_types.md")
  <br>
  Проверка того, что недавние изменения в коде (исправления, новые функции) работают корректно и не сломали существующий
  функционал.

  Примеры: Regression Testing, Re-testing / Confirmation Testing, Migration Testing.

---

### 3. УРОВНИ ДОСТУПА К КОДУ / СИСТЕМЕ (System Knowledge Levels)

- `Black Box` (Черный ящик): Тестирование без знания внутреннего устройства системы, основанное на требованиях и
  спецификациях.
- `White Box` (Белый ящик): Тестирование с полным знанием исходного кода (`source code`) и внутренней структуры системы.
- `Grey Box` (Серый ящик): Тестирование с частичным знанием внутреннего устройства, часто на уровне взаимодействия
  модулей или `API`.

---

### 4. УРОВНИ ИСПОЛНЕНИЯ КОДА (Test Execution Levels)

- [`Static Testing` (Статическое тестирование)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Static_testing.md "Static_testing.md")
  <br>
  Анализ артефактов (код, требования, дизайн) без запуска программы.
- [`Dynamic Testing` (Динамическое тестирование)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Dynamic_testing.md "Dynamic_testing.md")
  <br>
  Тестирование, при котором программный код исполняется для проверки его реального поведения при заданных входных
  данных.

---

### 5. СТЕПЕНИ АВТОМАТИЗАЦИИ ТЕСТИРОВАНИЯ (Test Automation Levels)

> Классификация по способу выполнения тестов (независима от функциональной принадлежности).

- `Manual Testing` (Ручное тестирование): Выполнение тестовых сценариев (`test scenarios`) человеком без использования
  скриптов.
- `Automated Testing` (Автоматизированное тестирование): Выполнение тестов с помощью специальных программ и скриптов.

---

> **Примечание по пересечению категорий**
>
> Типы тестирования могут комбинироваться с уровнями и подходами. Например:
> - `Regression Testing` (Change-related) может проводиться на уровне `Unit`, `Integration` или `System`.
> - `Security Testing` (Non-functional) может выполняться как `Black Box` (пентест) или `White Box` (SAST).
> - `Automated Functional Tests` часто реализуются как `Integration` или `E2E` тесты.

---

## ПРОЦЕССЫ И АРТЕФАКТЫ ТЕСТИРОВАНИЯ

- [`STLC` (Software testing lifecycle)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/STLC.md "STLC.md")
- [`Test Documentation And Metrics` (Тестовая документация и метрики)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Test_documentation.md "Test_documentation.md")
- [`Defects and Errors` (Дефекты и ошибки)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Defects_and_Errors.md "Defects_and_Errors.md")

---

## МЕТОДОЛОГИИ И ТЕХНИКИ ТЕСТИРОВАНИЯ

- [`Test Design technics` (Техники Тест-Дизайна)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Test_design_technics.md "Test_design_technics.md")

- [`Risk-based Testing` (`RBT`) (Тестирование на основе рисков)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Risk-based_testing.md "Risk-based_testing.md")
  <br>
  Методология, при которой объем, глубина и приоритеты тестирования определяются на основе анализа потенциальных рисков
  для продукта и проекта.

- [`Root Cause Analysis` (`RCA`) (Анализ первопричин)](https://github.com/sergeev-alexander/sergeev-alexander/blob/main/tech_stack/testing/Root_cause_analysis.md "Root_cause_analysis.md")
  <br>
  Систематический подход к выявлению фундаментальных (корневых) причин инцидентов, дефектов или сбоев с целью их
  устранения и предотвращения повторного возникновения.

---