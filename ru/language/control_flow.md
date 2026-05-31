# XCX 3.1 — управление потоком

## Условные операторы (if/elseif/else)

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**Синонимы:**

| Ключевое слово | Синонимы       |
|----------------|----------------|
| `elseif`       | `elif`, `elf`  |
| `else`         | `els`          |

Все формы эквивалентны и могут смешиваться:

```xcx
i: score = 75;
if (score >= 90) then;
    >! "A";
elif (score >= 75) then;
    >! "B";
elf (score >= 60) then;
    >! "C";
els;
    >! "F";
end;
```

---

## Циклы

### Цикл for по диапазону

Диапазон **включительный** с обеих сторон.

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- prints: 1, 2, 3
```

### Цикл for с шагом (`@step`)

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- prints: 0, 2, 4, 6
```

### Перебор коллекций

Работает с массивами, множествами и файберами. Переменная цикла получает **элемент**, а не индекс.

```xcx
--- Array
array:i: nums {10, 20, 30};
for el in nums do;
    >! el;
end;

--- Set
set:N: primes {2, 3, 5, 7, 11};
for p in primes do;
    >! p;
end;

--- Fiber
fiber:i: f = gen(5);
for val in f do;
    >! val;
end;
```

### Цикл while

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### break и continue

`break` выходит из текущего цикла. `continue` переходит к следующей итерации. Оба действуют только на **ближайший охватывающий цикл**.

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> `break` внутри цикла `for` по файберу автоматически вызывает `.close()` у этого файбера.
