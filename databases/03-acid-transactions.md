# 3. ACID in a transaction

> **Question:** "What does ACID mean in a transaction?"

---

## 🗣️ Answer (say it out loud, 1–2 min)

> ACID is the set of guarantees a database gives you for a **transaction**, which is a group of operations treated as a single unit.
>
> - **Atomicity** means **all or nothing**. Either every operation in the transaction is applied, or none of them is. If something fails in the middle, the database **rolls back** to the state before the transaction.
> - **Consistency** means a transaction takes the database **from one valid state to another**. All the rules, like primary keys, foreign keys, unique and check constraints, must hold at the end.
> - **Isolation** means concurrent transactions **don't see each other's partial work**. How strict that is depends on the **isolation level**: in MySQL InnoDB the default is **Repeatable Read**.
> - **Durability** means once a transaction is **committed**, it stays committed, even if the server crashes right after. InnoDB writes the change to the **redo log** on disk before confirming the commit.
>
> The classic example is a **bank transfer**: I subtract 100 from account A and add 100 to account B. Those two updates must succeed or fail **together**. If the server crashes after the first update, atomicity guarantees the money doesn't just disappear.

---

## Explicación (en español)

| Letra | Significa | Qué lo garantiza en MySQL InnoDB | Si no existiera... |
|---|---|---|---|
| **A**tomicity | Todo o nada | **Undo log** + `ROLLBACK` | Se descuenta de A pero no se acredita en B |
| **C**onsistency | De un estado válido a otro válido | Constraints (PK, FK, `UNIQUE`, `CHECK` desde 8.0.16) + la lógica de tu app | Quedan pedidos que apuntan a clientes que no existen |
| **I**solation | Las transacciones concurrentes no se pisan | Locks + **MVCC** (cada transacción ve un snapshot) | Dos personas compran el último asiento |
| **D**urability | Lo confirmado no se pierde | **Redo log** en disco (`innodb_flush_log_at_trx_commit = 1`) | Un corte de luz borra pagos ya confirmados |

> 💡 Matiz que suma puntos: **Consistency** es la única que **no depende solo de la base de datos**. La BD hace cumplir las constraints, pero las reglas de negocio ("el saldo no puede ser negativo") también las tiene que respetar tu código o expresarlas como constraint.

### Niveles de aislamiento (la pregunta de seguimiento más común)

| Nivel | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Posible | Posible | Posible |
| Read Committed | ❌ | Posible | Posible |
| **Repeatable Read** (default MySQL) | ❌ | ❌ | En InnoDB, mitigado en gran parte con *next-key locks* |
| Serializable | ❌ | ❌ | ❌ |

- **Dirty read**: leer un cambio que otra transacción **todavía no confirmó** (y que quizás revierta).
- **Non-repeatable read**: lees la misma fila dos veces en tu transacción y obtienes valores distintos.
- **Phantom read**: repites la misma query y aparecen **filas nuevas**.

> Más aislamiento = más seguridad, pero **menos concurrencia** (más bloqueos y esperas). Es un trade-off.

---

## Ejemplo real (MySQL + .NET)

**Escenario:** una transferencia entre cuentas de una billetera digital.

### En SQL

```sql
START TRANSACTION;

-- FOR UPDATE bloquea las filas para que otra transferencia concurrente no lea un saldo viejo
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;   -- cuenta A: 500
SELECT balance FROM accounts WHERE id = 2 FOR UPDATE;   -- cuenta B

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

INSERT INTO transfers (from_account, to_account, amount, created_at) VALUES (1, 2, 100, NOW());

COMMIT;   -- si algo falla antes de esta línea → ROLLBACK y ninguna cuenta cambia
```

Y la regla de negocio como constraint, para que la **C** no dependa solo del código:

```sql
ALTER TABLE accounts ADD CONSTRAINT chk_balance_non_negative CHECK (balance >= 0);
```

### En .NET con EF Core

```csharp
public async Task TransferAsync(long fromId, long toId, decimal amount, CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);

    // Orden fijo de bloqueo (id menor primero) para evitar deadlocks entre transferencias A→B y B→A
    var ids = new[] { fromId, toId }.Order().ToArray();
    var accounts = await db.Accounts
        .FromSql($"SELECT * FROM accounts WHERE id IN ({ids[0]}, {ids[1]}) ORDER BY id FOR UPDATE")
        .ToDictionaryAsync(a => a.Id, ct);

    var from = accounts[fromId];
    var to   = accounts[toId];

    if (from.Balance < amount) throw new InsufficientFundsException();

    from.Balance -= amount;
    to.Balance   += amount;
    db.Transfers.Add(new Transfer(fromId, toId, amount, DateTime.UtcNow));

    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);    // si hay una excepción antes, el `await using` hace rollback
}
```

**Por qué cada detalle importa:**
- **Atomicity**: los dos `UPDATE` y el `INSERT` van en la misma transacción.
- **Isolation**: `FOR UPDATE` evita que dos transferencias simultáneas lean el mismo saldo y lo gasten dos veces.
- **Orden fijo de bloqueo**: si una transferencia bloquea A y luego B, y otra bloquea B y luego A, se produce un **deadlock**. Bloquear siempre en el mismo orden lo evita.
- **Consistency**: el `CHECK` impide saldos negativos aunque haya un bug en el código.
- **Durability**: una vez que `CommitAsync` termina, el cambio sobrevive a una caída del servidor.

> 🔗 Conexión con tu historia de S3: en el código de ejemplo del job de migración (el ilustrativo, no el del proyecto), **borrar el lote de MySQL y avanzar el checkpoint** van en una sola transacción. Si se hace solo una de las dos, se pierden o se duplican datos. Es atomicidad aplicada.

---

## Useful vocabulary

| English | Español |
|---|---|
| to commit / to roll back | confirmar / revertir |
| all or nothing | todo o nada |
| concurrent transactions | transacciones concurrentes |
| lock / row-level lock | bloqueo / bloqueo a nivel de fila |
| deadlock | interbloqueo |
| isolation level | nivel de aislamiento |
| crash | caída del servidor |

---

## Follow-up questions (practice in English)

1. What are dirty reads, non-repeatable reads and phantom reads?
2. What is MySQL's default isolation level? What would you change it to, and why?
3. What is a deadlock? How do you prevent it and how does MySQL handle it? (detects it and rolls back one transaction)
4. What's the difference between optimistic and pessimistic locking? (version column vs `SELECT ... FOR UPDATE`)
5. How do you keep consistency across **two different systems**, like MySQL and S3, where you can't use one transaction? (outbox pattern, retries, idempotency, saga)
6. What is the CAP theorem, and how is it different from ACID?
