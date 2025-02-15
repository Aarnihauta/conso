# 1. Спроектировать таблицы и связи между ними. 
```sql
create extension if not exists pg_trgm;
create table departments (
   id serial primary key,
   name varchar(255) not null
); --не уверен что тут требуется индекс по аналогии с employees

create table employees (
   id serial primary key,
   name varchar(255) not null
);
create index idx_employees_name_trgm on employees using GIN(name gin_trgm_ops); --для поиска по name в формате like %%

create table employee_department (
   employee_id int references employees(id),
   department_id int references departments(id),
   primary key (employee_id, department_id)
);

create table department_directors (
   department_id int primary key references departments(id),
   employee_id int references employees(id),
   start_date date not null
);
create index idx_department_directors_employee_id on department_directors(employee_id);
```

### a. Вывести всех сотрудников и их руководителей, и отделы, в которых работает сотрудник
```sql
select e.id as employee_id,
   e.name as employee_name,
   d.id as department_id,
   d.name as department_name,
   dd.employee_id as director_iod,
   m.name AS director_name
from employees e
join employee_department ed ON e.id = ed.employee_id
join departments d ON ed.department_id = d.id
left join department_directors dd ON d.id = dd.department_id
left join employees m ON dd.employee_id = m.id;```

### b. Вывести все отделы и количество сотрудников в отделе без руководителя 
```sql
select d.id as departtment_id,
   d.name as department_name,
   count(ed.employee_id) AS employee_count
from departments d
left join employee_department ed on d.id = ed.department_id
left join department_directors dd on d.id = dd.department_id
where dd.employee_id is null
group by d.id
```
### c. Вывести список ID отделов, количество сотрудников в которых не превышает 3 человек
```sql
select
      d.id as department_id
from departments d
left join employee_department ed on d.id = ed.department_id
group by d.id
having count(ed.employee_id) <= 3
```

### d. Вывести сотрудников, у которых стаж руководителя более 10 лет в компании
```sql
select e.id,
      e.name
from employees e
join employee_department ed on e.id = ed.employee_id
join department_directors dd on ed.department_id = dd.department_id
join employees m on dd.employee_id = m.id
where dd.start_date <= current_date - interval '10 years'
```

### e. Вывести список отделов, отсортировав по наименованию отдела, где первыми должны быть отделы, у которых в названии вторая буква «А», остальные в обратном порядке
```sql
select name
from departments
order by
   case
       when lower(name) like '_а%' then 0 --в ТЗ «А» - кириллица, поэтому сделал проверку только для неё. В ТЗ не написано, но я не учитываю регистр т.к отдел может являться аббревиатурой
       else 1
   end,
   name desc
```

# 5. Какие значения будут в переменных после выполнения следующего кода
@a = NULL, @b = 0

# 7. На C# написать простейший Singleton
"Простейший" в моём понимании выглядит вот так: `serviceCollection.AddSingleton<Singleton>()`. Но я так же напишу и потокобезопасную реализацию через Lazy, хоть по ТЗ и не требуется потокобезопасность
```csharp
public sealed class Singleton
{
    private static readonly Lazy<Singleton> _instance = new(new Singleton());

    private Singleton() { }

    public static Singleton Instance => _instance.Value;
}
```

# 8. Каково назначение ключевого слова lock в C#
lock это сахар для Monitor. `Monitor` в свою очередь нужен чтобы гарантировать выполнение только 1 потоку внутри тела. Когда выполняется `Monitor.Enter`, CLR запоминает текущий поток в заголовке объекта (managed object header) который передается в `Enter`, а когда вызывается метод `Monitor.Exit`, CLR проверяет, совпадает ли поток, который заблокировал объект, с потоком, который пытается разблокировать объект. Эта проверка как раз и гарантирует то, что только один поток может выполнять код в блоке. По этой же причине внутри Monitor (и следовательно в lock) нельзя использовать асинхронный код. Еще интересно, что подобный код приводит к блокировкам (т.к `"namedlock"` в заголовках PE-файлов будет иметь одну единственную ссылку, аналогичное хранение и у `const`):
```csharp
//Класс A
lock("namedlock")
{
}
//Класс B
lock("namedlock")
{
}
```
