# Blockchain — EVM

## Что такое EVM

**EVM (Ethereum Virtual Machine)** — децентрализованная виртуальная машина, обеспечивающая выполнение смарт-контрактов в блокчейне Ethereum. Работает одновременно на тысячах узлов по всему миру — не существует как физическое устройство.

**Свойства:**
- **Детерминированность** — одинаковые входные данные всегда дают одинаковый результат на всех нодах
- **Изолированность** — смарт-контракт не может напрямую читать состояние другого контракта без вызова
- **Тьюринг-полнота** — может исполнять произвольный код (ограничение — газ)
- **Защита от цензуры** — код выполняется точно, как написан, без возможности внешнего вмешательства

---

## EVM как машина состояний

```
State(n) ──[ Block с транзакциями ]──► State(n+1)
```

Ethereum — это **State Machine**: каждый блок переводит глобальное состояние из одного в другое. Состояние — это совокупность всех аккаунтов, балансов и хранилищ смарт-контрактов.

**Компоненты EVM:**
- **Stack** — 1024 элемента по 32 байта, основная рабочая память
- **Memory** — временная память в рамках одного вызова (очищается после)
- **Storage** — постоянное хранилище контракта (дорогое: SSTORE ~20000 gas)
- **Calldata** — входные данные транзакции (read-only, дёшево)

---

## Смарт-контракты

**Смарт-контракт** — программа, хранящаяся в блокчейне. Написана на Solidity → компилируется в **байткод** (EVM-инструкции) → деплоится транзакцией → исполняется при вызове.

```
Solidity код
    └─► solc (компилятор)
        ├── bytecode  → хранится в блокчейне (код контракта)
        └── ABI       → интерфейс для вызовов (JSON-описание функций)
```

**Типы аккаунтов Ethereum:**

| | EOA (Externally Owned Account) | Contract Account |
|---|---|---|
| Управление | Приватным ключом | Кодом контракта |
| Код | Нет | Есть (bytecode) |
| Инициация | Может начинать tx | Только в ответ на вызов |
| Storage | Нет | Есть |

---

## Gas

**Gas** — единица измерения вычислительной работы в EVM. Каждая операция байткода стоит определённое количество газа.

**Зачем нужен:**
- Предотвращает бесконечные циклы (Halting Problem)
- Компенсирует ноды за вычисления
- Создаёт рынок приоритета транзакций

```
Transaction Cost = Gas Used × Gas Price (в Gwei)
                 = Gas Used × (Base Fee + Priority Fee)
```

**EIP-1559 (после London Upgrade):**

| Поле | Описание |
|---|---|
| `baseFee` | Минимальная плата, устанавливается протоколом (сжигается) |
| `maxPriorityFeePerGas` | Чаевые валидатору (tip) |
| `maxFeePerGas` | Максимум, который согласен платить пользователь |

```
Реальная комиссия = min(baseFee + tip, maxFeePerGas) × gasUsed
```

**Стоимость операций (примеры):**

| Операция | Gas |
|---|---|
| ADD, SUB | 3 |
| MUL, DIV | 5 |
| SLOAD (чтение storage) | 2100 (холодный) / 100 (тёплый) |
| SSTORE (запись storage) | ~20000 (новое) / ~5000 (обновление) |
| Перевод ETH | 21000 |
| Деплой контракта | ~32000 + 200 × bytes |

**Gas Limit:**
- `gasLimit` — максимум, который транзакция может потратить
- Если газ кончается до завершения — `out of gas`, транзакция откатывается, газ не возвращается

---

## Статусы транзакций

| Статус | Описание |
|---|---|
| **Pending** | В mempool, ожидает включения в блок |
| **Success** | Выполнена, включена в блок, данные необратимы |
| **Failed** | Ошибка исполнения (out of gas, revert), изменения отменены, газ не возвращается |
| **Dropped** | Удалена из mempool (слишком старая, вытеснена с тем же nonce) |
| **Replaced** | Заменена новой транзакцией с тем же nonce, но большим gas price |

---

## ABI и вызов функций

**ABI (Application Binary Interface)** — JSON-описание интерфейса смарт-контракта. Позволяет внешнему коду кодировать/декодировать вызовы.

```json
{
    "name": "transfer",
    "type": "function",
    "inputs": [
        {"name": "to", "type": "address"},
        {"name": "amount", "type": "uint256"}
    ],
    "outputs": [{"name": "", "type": "bool"}]
}
```

**Calldata кодирование:**
```
transfer(address to, uint256 amount)
    → selector: keccak256("transfer(address,uint256)")[0:4]  = 0xa9059cbb
    → encoded:  0xa9059cbb + padded(to) + padded(amount)
```

```ts
// ethers.js — взаимодействие с контрактом
import { ethers } from 'ethers';

const provider = new ethers.JsonRpcProvider('https://mainnet.infura.io/v3/KEY');
const contract = new ethers.Contract(address, abi, provider);

// Чтение (view/pure — бесплатно, не создаёт транзакцию)
const balance = await contract.balanceOf(userAddress);

// Запись (создаёт транзакцию, тратит газ)
const signer = new ethers.Wallet(privateKey, provider);
const contractWithSigner = contract.connect(signer);
const tx = await contractWithSigner.transfer(toAddress, amount);
const receipt = await tx.wait();  // ждём включения в блок
console.log(receipt.status);  // 1 = success, 0 = failed
```

---

## Events и Logs

**Events** — способ эмитировать данные из смарт-контракта в логи блокчейна. Дешевле хранения в storage (~375 gas за topic).

```solidity
// Solidity
event Transfer(address indexed from, address indexed to, uint256 value);

// Внутри функции:
emit Transfer(msg.sender, to, amount);
```

```ts
// Чтение событий через ethers.js
const filter = contract.filters.Transfer(null, userAddress);  // indexed param
const events = await contract.queryFilter(filter, fromBlock, toBlock);

// Подписка на события реального времени
contract.on('Transfer', (from, to, value, event) => {
    console.log(`Transfer: ${from} → ${to}: ${value}`);
});
```

**`indexed`** параметры сохраняются в topics (до 3 штук) и по ним можно фильтровать. Неиндексированные — в data (дешевле, но не фильтруются).

---

## Паттерны смарт-контрактов

**Proxy Pattern (обновляемые контракты):**
```
User → Proxy Contract → Implementation Contract
         (хранит state)    (содержит логику)
```
Proxy делегирует вызовы через `delegatecall` — код выполняется в контексте proxy (использует его storage). При обновлении меняется только адрес Implementation.

**Reentrancy Guard:**
```solidity
// Уязвимость: рекурсивный вызов до обновления состояния
// Защита: Checks-Effects-Interactions pattern
function withdraw(uint amount) external {
    require(balances[msg.sender] >= amount);   // Check
    balances[msg.sender] -= amount;             // Effect (СНАЧАЛА обновить state)
    (bool success,) = msg.sender.call{value: amount}("");  // Interaction
    require(success);
}
```

**Multisig:** транзакция выполняется только при подписи N из M ключей (например, 2 из 3).
