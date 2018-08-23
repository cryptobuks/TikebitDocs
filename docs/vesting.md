# Vesting

En este contrato se asignarán a cada dirección las cantidades de tokens [TKB](/tkb/) y fecha de retirada que se hayan pactado previamente. Este contrato está destinado a guardar y congelar los tokens destinados al equipo, advisors, private sale, etc. La introducción de direcciones solo puede hacerse desde la dirección de `owner`, pero solo las direcciones apuntadas podrán retirar fondos una vez se haya pasado el tiempo necesario

## Variables
----

- **amountWithdrawn** `uint256` `public` `constant`
- **TKBabi** `TKB` `public`
- **categories** `Category[]` `public`

## Structs
----

Cada `Category` representa los diferentes tipos de grupos que recibirán tokens. 
```
struct Category {
    string name;
    uint256 amount;
    uint256 filledAmount;
    uint256 withdrawDate;
    mapping(address => uint) participantAmount;
}
```

## Mappings
----

Guarda el `index` de la `Category` a la que está asignado.
``` 
mapping(address => uint) userCategory;
```
Guarda las direcciones que ya han retirado su tokens.
```
mapping(address => bool) alreadyWithdrawn;
```

## Funciones
----
### constructor()
En el momento de la creación del contrato, se generarán las `Category` que se especifiquen.
```
constructor(address _owner) 
    public 
{
    categories.push(Category("Private sale", 50000000*1000000000000000000, 0, block.timestamp.add(24 weeks)));
    categories.push(Category("Pre sale", 100000000*1000000000000000000, 0, block.timestamp.add(24 weeks)));
    categories.push(Category("Team", 200000000*1000000000000000000, 0, block.timestamp.add(48 weeks)));
    categories.push(Category("Advisors", 30000000*1000000000000000000, 0, block.timestamp.add(24 weeks)));
    categories.push(Category("Partnerships", 30000000*1000000000000000000, 0, block.timestamp.add(12 weeks)));
    }
```

### addAddress()
Añade una dirección a la `Category` que se indique en `_index` y se le asigna la cantidad especificada en `_amount`.
Primero se comprueba que la dirección no sea 0x000..., que el `_index` especificado es valido y si el `_amount` añadido al total de la categoría, no pasa el limite de dicha `Category`.
Despues se incrementa el `filledAmount` de la `Category` indicada y se le asigna el `_amount` en el mapping `participantAmount`. 
```
function addAddress(
    address _who, 
    uint256 _amount, 
    uint256 _index
) 
    public 
    onlyOwner 
{
    require(_who != address(0), "Address can't be 0x0000...");
    require(_index >= 0 && _index < categories.length.sub(1), "Index has to be valid");
    require(_amount != 0, "Amount must be more than 0");
    require(_amount.add(categories[_index].filledAmount) < categories[_index].amount, "Amount exceeds the limit.");

    categories[_index].filledAmount = categories[_index].filledAmount.add(_amount);
    categories[_index].participantAmount[_who] = categories[_index].participantAmount[_who].add(_amount);
    userCategory[_who] = _index;
}
```
_Argumentos_


- **_who** `string` Dirección que se va a añadir.
- **_amount** `uint256` Cantidad a añadir.
- **_index** `uint256` Índice de la categoría.

### withdraw()
Primero se comprueba que la dirección puede sacar ya los fondos, si ya ha retirado los fondos y si tiene algún token que poder retirar.
Despues, se le asigna que ya ha retirado los tokens, se incrementa `amountWithdrawn` y finalmente se le transfieren los tokens `TKB`.
```
function withdraw() 
    public 
{
    address _who = msg.sender;
    require(categories[userCategory[_who]].withdrawDate >= block.timestamp, "You can't withdraw yet.");
    require(!alreadyWithdrawn[_who], "Already withdrawn.");
    uint256 _amount = categories[userCategory[_who]].participantAmount[_who];
    require(_amount != 0, "You don't have any amount to withdraw");

    alreadyWithdrawn[_who] = true;
    amountWithdrawn = amountWithdrawn.add(_amount);
    TKBabi.transfer(_who, _amount);
}
```
_Argumentos_


- **_who** `string` Dirección que se va a retirar los fondos.

### getUserBalance()
Devuelve la cantidad de tokens que tiene asignado `_who`.
```
function getUserBalance(
    address _who
) 
    public 
    view 
    returns (uint) 
{
    return categories[userCategory[_who]].participantAmount[_who];
}
```
_Argumentos_


- **_who** `string` Dirección de la cual devolver el balance.


__Devuelve__


- `uint256` Balance del usuario.
