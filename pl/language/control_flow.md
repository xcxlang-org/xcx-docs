# XCX 3.1 Przepływ sterowania

## Instrukcje warunkowe (if/elseif/else)

```xcx
if (condition) then;
    --- block
elseif (other_condition) then;
    --- block
else;
    --- block
end;
```

**Aliasy:**

| Słowo kluczowe | Aliasy         |
|---------------|----------------|
| `elseif`      | `elif`, `elf`  |
| `else`        | `els`          |

Wszystkie formy są równoważne i można je mieszać:

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

## Pętle

### For z zakresem

Zakres jest **włączny** z obu stron.

```xcx
for i in 1 to 3 do;
    >! i;
end;
--- prints: 1, 2, 3
```

### For z krokiem (`@step`)

```xcx
for j in 0 to 6 @step 2 do;
    >! j;
end;
--- prints: 0, 2, 4, 6
```

### Iteracja po kolekcji

Działa na tablicach, zbiorach i włóknach. Zmienna pętli otrzymuje **element**, a nie indeks.

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

### Pętla while

```xcx
i: cnt = 0;
while (cnt < 3) do;
    cnt = cnt + 1;
    >! cnt;
end;
```

### Break i continue

`break` kończy bieżącą pętlę. `continue` przechodzi do następnej iteracji. Oba wpływają wyłącznie na **bezpośrednio otaczającą pętlę**.

```xcx
for n in 1 to 5 do;
    if (n % 2 == 0) then; continue; end;
    if (n == 5) then; break; end;
    >! n;
end;
```

> [!NOTE]
> `break` wewnątrz pętli `for` po włóknach automatycznie wywołuje `.close()` na tym włóknem.
