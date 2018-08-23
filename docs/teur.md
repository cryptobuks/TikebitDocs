# TEUR - Tikebit Euro

La funcionalidad del TEUR es sencilla, ya que es un token ERC20 con todas las funcionalidades de las interfaces [ERC20Basic](/interface/) y [ERC20](/interface/), además de incluir **mint** y **burn**. 

## Variables
----

- **name** `address` `public` `constant`
- **symbol** `address` `public` `constant`
- **decimals** `uint8` `public` `constant`
- **totalSupply_** `uint256` `internal`
- **TKBabi** `TKB` `public`


## Mappings
----

El mapping de "balances" es el que controla la cantidad de tokens que tiene cada dirección.
``` 
mapping(address => uint256) balances;
```
El mapping de "allowed" guarda la cantidad de tokens que cada dirección ha concedido a otra dirección.
```
mapping(address => mapping(address => uint256)) internal allowed;
```

## Eventos
----

```
event Burn(address indexed burner, uint256 value);
event Mint(address indexed to, uint256 amount);
event TKBfee(uint256 amount);
```

## Funciones
----
### constructor()
En el momento de la creación del contrato, al ser desplegado por el smart contract TKB, teniendo la ABI ya, vinculamos el `msg.sender` al ABI y también le transfiere los permisos a la dirección de `_owner`.
```
constructor(address _owner) 
    public 
{
    TKBabi = TKB(msg.sender);
    Ownable.transferOwnership(_owner);
    Ownable.transferSuperUser(_owner);
}
```
_Argumentos_


- **_owner_** `address` Dirección del nuevo `owner` y `superUser`.

### burn()
Al usar esta función, se mandará un X% a el contrato TKB.
Comprueba que `_amount` no es 0 y que el `msg.sender` tiene suficiente balance. Después calcula y asigna el % correspondiente al TKB y después resta el `_amount` al `msg.sender` y resta de `totalSupply_` el restante del `_amount` sin la cantidad enviada al TKB.
```
function burn(
    uint256 _amount
) 
    public
    onlyOwner 
    returns (bool)
{
    require(_amount > 0, "The value must be more than 0");
    require(_amount <= balances[msg.sender], "Insufficient balance.");
    address _who = msg.sender;

    uint256 _amountForTKB = _amount.mul(150).div(10000);
    balances[TKBabi] = balances[TKBabi].add(_amountForTKB);
    uint256 _amountWithoutFee = _amount.sub(_amountForTKB);

    balances[_who] = balances[_who].sub(_amountWithoutFee);
    totalSupply_ = totalSupply_.sub(_amountWithoutFee);
    emit Burn(_who, _amountWithoutFee);
    return true;
}
```
_Argumentos_

- **_amount** `uint256` Cantidad de tokens.


_Devuelve_


- `bool` Comprobación de que ha acabado la función.

### mint()
Al usar esta función, se mandará un X% a el contrato TKB y se quitará un Y% correspondiente a la comisión de la tienda que creó el tiket.
Comprueba que `_amount` no es 0, que `_to` no sea 0x000... y que `_fee` es mayor que 0 y menor que 500. Después, calcula y quita la cantidad que se queda el CashNode y calcula la cantidad que se va al TKB y se lo asigna. Una vez ya se han sacado todas las comisiones, se aumenta el `totalSupply_` y se le asignan los tokens restantes de `_amount`.
```
function mint(
    address _to,
    uint256 _amount,
    uint256 _fee
)
    public
    onlyOwner
    returns (bool)
{
    require(_amount > 0, "Amount must be more than 0");
    require(_to != address(0), "You can't send tokens to this address");
    require(_fee > 0, "Fees must be more than 0");
    require(_fee < 500, "Fees must be lower than 5%-500 units");

    uint256 _amountCashNodeFee = _amount.mul(_fee).div(10000);
    uint256 _amountWithoutFee = _amount.sub(_amountCashNodeFee);

    totalSupply_ = totalSupply_.add(_amount);

    uint256 _amountForTKB = _amount.mul(150).div(10000);
    _amountWithoutFee = _amount.sub(_amountForTKB);

    balances[_to] = balances[_to].add(_amountWithoutFee);
    balances[TKBabi] = balances[TKBabi].add(_amountForTKB);

    emit Mint(_to, _amountWithoutFee);
    emit TKBfee(_amountForTKB);
    emit Transfer(address(0), _to, _amountWithoutFee);
    return true;
}
```
_Argumentos_

- **_amount** `uint256` Cantidad de tokens a crear.
- **_fee** `uint256` Cantidad de tokens que se queda el CashNode.
- **_to** `address` Dirección a la que enviar los tokens.


_Devuelve_


- `bool` Comprobación de que ha acabado la función.

### totalSupply()
Devuelve el total de tokens que hay. (Devuelve el `totalSupply_`)
```
function totalSupply() 
    public 
    view 
    returns(uint256) 
{
    return totalSupply_;
}
```
_Devuelve_


- `uint256` Cantidad de tokens que existen.

### transfer()
Comprueba que `_to` no es la dirección 0x000... y que el `msg.sender` tiene suficiente balance. Después realiza la transferencia y emite el evento.
```
function transfer(
    address _to, 
    uint256 _value
) 
    public 
    returns(bool) 
{
    require(_to != address(0), "Address can't be 0x0000...");
    require(_value <= balances[msg.sender], "Insufficient balance");
    balances[msg.sender] = balances[msg.sender].sub(_value);
    balances[_to] = balances[_to].add(_value);
    emit Transfer(msg.sender, _to, _value);
    return true;
}
```
_Argumentos_


- **_to** `address` Dirección del destinatario.
- **_value** `uint256` Cantidad a transferir.

### balanceOf()
Devuelve la cantidad de tokens que tiene `_owner`.
```
function balanceOf(address _owner) 
    public 
    view 
    returns(uint256) 
{
    return balances[_owner];
}
```
_Argumentos_


- **_owner** `address` Dirección de quien devolver el balance.

### transferFrom()
Comprueba que `_to` no es la dirección 0x000..., que `_from` tenga suficiente balance y que `_from` le haya concedido suficiente balance a `msg.sender`. Después reduce el balance de `_from`, incrementa el balance de `_to` y reduce el `allowance` de `_from` a `msg.sender`.
```
function transferFrom(
    address _from,
    address _to,
    uint256 _value
)
    public
    returns(bool) 
{
    require(_to != address(0), "Adress can't be 0x0000...");
    require(_value <= balances[_from], "Insufficient balance.");
    require(_value <= allowed[_from][msg.sender], "Insufficient balance allowed.");

    balances[_from] = balances[_from].sub(_value);
    balances[_to] = balances[_to].add(_value);
    allowed[_from][msg.sender] = allowed[_from][msg.sender].sub(_value);
    emit Transfer(_from, _to, _value);
    return true;
    }
```
_Argumentos_


- **_from** `address` Dirección de la que sacar los tokens.
- **_to** `address` Dirección a quien enviar los tokens.
- **_value** `uint256` Cantidad de tokens a sacar.

_Devuelve_


- `bool` Comprobación de que ha acabado la función.

### approve()
El `msg.sender` concede a `_spender` la cantidad de tokens `_value`.
```
function approve(
    address _spender, 
    uint256 _value
) 
    public 
    returns(bool) 
{
    allowed[msg.sender][_spender] = _value;
    emit Approval(msg.sender, _spender, _value);
    return true;
}
```
_Argumentos_


- **_spender** `address` Dirección a quien ceder los tokens.
- **_value** `uint256` Cantidad de tokens a ceder.

_Devuelve_


- `bool` Comprobación de que ha acabado la función.

### allowance()
Devuelve cuantos tokens le ha cedido `_owner` a `_spender`.
```
function allowance(
    address _owner,
    address _spender
)
    public
    view
    returns(uint256) 
{
    return allowed[_owner][_spender];
}
```
_Argumentos_

- **_owner** `address` Dirección que cede los tokens
- **_spender** `address` Dirección a quien han cedido tokens.

_Devuelve_


- `uint256` Cantidad cedida.

### increaseApproval()
Incrementa la cantidad cedida de `msg.sender` a `_spender`. La cantidad se indica en `_addedValue`.
```
function increaseApproval(
    address _spender,
    uint256 _addedValue
)
    public
    returns(bool) 
{
    allowed[msg.sender][_spender] = (
        allowed[msg.sender][_spender].add(_addedValue));
    emit Approval(msg.sender, _spender, allowed[msg.sender][_spender]);
    return true;
}
```
_Argumentos_

- **_spender** `address` Dirección a quien han cedido tokens.
- **_addedValue** `uint256` Cantidad a aumentar la cesión.


_Devuelve_


- `bool` Comprobación de que ha acabado la función.

### decreaseApproval()
Reduce la cantidad cedida de `msg.sender` a `_spender`. La cantidad se indica en `_subtractedValue`.
```
function decreaseApproval(
    address _spender,
    uint256 _subtractedValue
)
    public
    returns(bool) 
{
    uint256 oldValue = allowed[msg.sender][_spender];
    if (_subtractedValue > oldValue) {
        allowed[msg.sender][_spender] = 0;
    } else {
        allowed[msg.sender][_spender] = oldValue.sub(_subtractedValue);
    }
    emit Approval(msg.sender, _spender, allowed[msg.sender][_spender]);
    return true;
}
```
_Argumentos_

- **_spender** `address` Dirección a quien han cedido tokens.
- **_subtractedValue** `uint256` Cantidad a reducir la cesión.


_Devuelve_


- `bool` Comprobación de que ha acabado la función.

