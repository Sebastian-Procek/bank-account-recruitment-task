# Bank Account — DDD Code Sample

A small, **framework-agnostic** PHP domain model implemented with **Domain-Driven Design** tactical patterns.
Built as a recruitment task, but written the way I'd model a real domain: rich behaviour in the model,
business rules isolated behind policies, and full unit-test coverage.

> **Stack:** PHP 8.x · PHPUnit 11 · `webmozart/assert` · PSR-4 · zero frameworks

---

## Domain

A bank account that:

1. Has a single assigned **currency**.
2. Supports **credit** (receiving) and **debit** (sending) money in that currency.
3. Computes its **balance** from the transaction history.
4. Charges a **0.5% fee** on every debit.
5. Allows a debit only when there are **sufficient funds**.
6. Allows at most **3 debits per day**.

## Design

The model is intentionally **rich** — invariants live inside the domain, not in services.

| Pattern | Where | Why |
|---------|-------|-----|
| **Aggregate root** | `Account` | Single entry point for state changes; private constructor + `Account::openAccount()` named constructor guard creation. |
| **Value Objects** | `Money`, `Balance` | Immutable (`readonly`), self-validating, currency-safe arithmetic (`Money::add()`). |
| **Entity** | `Transaction` | Records an operation + amount + timestamp. |
| **Enums** | `Currency`, `Operation` | Type-safe domain vocabulary. |
| **Policy / Strategy** | `DebitCostsPolicy`, `DebitLimitsPolicy`, `DebitOverdraftPolicy` | Each business rule is a swappable strategy behind an interface — fees, daily limits and overdraft checks can change without touching the aggregate. |
| **Domain exceptions** | `DomainException` + subtypes | Rule violations are explicit, typed failures (e.g. `BalanceAccountIsToLowException`, `DailyAccountDebitTransactionLimitExceededException`). |

**Dependency Inversion:** `Account` depends on policy *interfaces*, injected at construction — so the standard rules
(`StandardDebitCosts`, `StandardDebitLimits`, `StandardDebitOverdraft`) are just one implementation among many.

```
src/BankAccount/
└── Model/
    ├── Account.php            # aggregate root
    ├── Transaction.php        # entity
    ├── VO/                    # Money, Balance
    ├── Enum/                  # Currency, Operation
    ├── Exception/             # domain exceptions
    └── Policy/                # DebitCosts / DebitLimits / DebitOverdraft (interface + standard impl)
```

## Getting started

Requirements: **PHP 8.x** and **Composer**.

```bash
composer install
```

## Running the tests

```bash
./vendor/bin/phpunit
```

The suite (`tests/BankAccount/Unit`) covers the business scenarios — fee calculation, overdraft
protection, the daily debit limit and currency consistency. PHPUnit is configured strictly
(`failOnWarning`, `failOnRisky`, `failOnPhpunitDeprecation`).

## Usage

```php
use App\BankAccount\Model\Account;
use App\BankAccount\Model\Enum\Currency;
use App\BankAccount\Model\VO\Money;
use App\BankAccount\Model\Policy\DebitCosts\StandardDebitCosts;
use App\BankAccount\Model\Policy\DebitLimits\StandardDebitLimits;
use App\BankAccount\Model\Policy\DebitOverdraft\StandardDebitOverdraft;

$account = Account::openAccount(
    Currency::PLN,
    new StandardDebitCosts(),
    new StandardDebitLimits(),
    new StandardDebitOverdraft(),
);

$account->credit(new Money(10_000, Currency::PLN)); // deposit
$account->debit(new Money(2_000, Currency::PLN));   // withdraw + 0.5% fee

echo $account->currentBalance()->money->amount; // 7_990
```

> Money amounts are integers in the currency's **minor unit** (e.g. grosze / cents) to avoid floating-point errors.

---

*Author: [Sebastian Procek](https://github.com/Sebastian-Procek) · sebastian.procek@gmail.com*
