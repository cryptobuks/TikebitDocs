# TKB - Tikebit Coin

El TKB implementa las funcionalidades de `ERC20Basic` y `ERC20`. La finalidad de este token es permitir el reparto de las `fee` que se generan en el [TEUR](/teur/). Para conseguir esto, se podrán generar `stakes`, que contienen `stakeUnits` las cuales representan las unidades proporcionales que recibirás respecto a la `feePool`. También se implementa un sistema de fechas donde está todo controlado para que se restrinja cada acción a determinada fecha. Los tokens usados en los stakes se restarán del balance de la dirección en funciones como `transfer()`, `transferFrom()` y las que requieran saber del balance de tokens de una dirección.


Las transferencias estarán congeladas hasta que se termine la ICO, la variable que controla esto es `stopped`. Es posible que una vez terminada la ICO se vuelva a para el contrato, pero solo en el caso de que se pida un `refund` de la DAICO, permitiendo así ir al contrato de la DAICO a retirar ETH a razón de los tokens TKB que tengas. Esta variable no es posible cambiarla de forma manual, se descongelará una vez termine la ICO de forma automática y al pedir un `refund` mediante la DAO.

## Variables
----

- **name** `address` `public` `constant`
- **symbol** `address` `public` `constant`
- **decimals** `uint8` `public` `constant`
- **totalSupply_** `uint256` `internal`
- **totalFeeCollected** `uint256` `public`
- **totalFeeWithdrawn** `uint256` `public`
- **dateWeek** `uint256` `public`
- **pricePerStakeUnit** `uint256` `public` `constant`
- **amountStakeUnits** `uint256` `public`
- **stopped** `bool` `public`
- **DAICOaddress** `address` `public`
- **TEURaddress** `address` `public`
- **VestingAddress** `address` `public`
- **DAICOabi** `DAICO` `public`
- **TEURabi** `TEUR` `public`
- **VestingAbi** `Vesting` `public`

## Structs
----

Un `Stake` está asociado a una dirección, lo cual permite saber la última vez que retiró fees, cuantos tokens tiene en stake y tiene un array de `Units` que representa la cantidad de unidades que tiene en stake.
```
struct Stake {
    uint256 tokensStaked;
    uint256 dateLastWithdraw;
    Units[] stakeUnits;
}
```
Cuando hablamos de `Units`, se refiere a la cantidad de `pricePerStakeUnit` que se han puesto en stake. En este struct, contiene dos valores, la fecha en la que se podrá hacer `undoStake` y la cantidad de unidades que se utilizaron.
```
struct Units {
    uint256 dateToUnstake;
    uint256 stakedUnits;
}
```
## Mappings
----

El mapping de `balances` es el que controla la cantidad de tokens que tiene cada dirección.
``` 
mapping(address => uint256) balances;
```
El mapping de `allowed` guarda la cantidad de tokens que cada dirección ha concedido a otra dirección.
```
mapping(address => mapping(address => uint256)) internal allowed;
```
El mapping `stake` asigna una dirección a un struct `Stake`.
```
mapping(address => Stake) public stake;
```
Historial de cuanto `fee` se ha recaudado, asignando la fecha a la cantidad.
```
mapping(uint256 => uint256) internal feesPerWeek;
```

## Funciones
----
### constructor()
En el momento de la creación del contrato, se desplegará el contrato `TEUR`, haciendo eso, se guardará la dirección del contrato y se guardará la dirección asociada con la ABI, para poder llamar de forma más sencilla a ese contrato en el futuro.
```
constructor() 
    public 
{
    // DAICOaddress = new DAICO(msg.sender, address(this), block.timestamp.add(6 hours));
    // DAICOabi = DAICO(DAICOaddress);

    TEURaddress = new TEUR(msg.sender);
    TEURabi = TEUR(TEURaddress);

    // VestingAddress = new Vesting(msg.sender);
    // VestingAbi = Vesting(VestingAddress);

    // <-- AQUÍ HAY QUE DECLARAR MUCHAS VARIABLES Y DECIDIR EL REPARTO INICIAL DE TKB
}
```

### createStake()
La función está separada en dos para facilitar la auditabilidad del código, ya que en la primera parte se realizan todas las comprobaciones y en el segundo las acciones.
Primero se comprueban las fechas. La dirección tiene que retirar todas las `fees` hasta la fecha y no se puede hacer `stake` fuera de las horas indicadas. Después se calculan las cantidades y se comprueba que la dirección tenga suficiente balance. En la parte de la acción, se incrementan los `tokensStaked` de la dirección y se añade un struct `Units` al array de `stakeUnits`. 
```
function createStake(
    uint256 _unitsToStake
) 
    public 
{
    require(stake[msg.sender].dateLastWithdraw == dateWeek || stake[msg.sender].dateLastWithdraw == 0, "Withdraw previous fees before staking");
    require(block.timestamp >= dateWeek.add(2 hours) && block.timestamp <= dateWeek.add(1 days), "It's not time to stake yet");

    require(_unitsToStake > 0, "Units must be more than 1.");
    uint256 _amount = _unitsToStake.mul(pricePerStakeUnit);
    require(balanceOf(msg.sender) >= _amount, "Insufficient balance");
    _createStake(_amount, _unitsToStake, msg.sender);
}


function _createStake(
    uint256 _amountToStake, 
    uint256 _unitsToStake, 
    address _user
) 
    internal 
{

    stake[_user].tokensStaked = stake[_user].tokensStaked.add(_amountToStake);
    stake[_user].stakeUnits.push(Units(block.timestamp, _amountToStake));
    stake[_user].dateLastWithdraw = dateWeek.add(1 weeks);
    amountStakeUnits = amountStakeUnits.add(_unitsToStake);
}
```
_Argumentos_


- **_unitsToStake** `uint256` Cantidad de unidades a incrementar el stake.

### undoStake()
La función está separada en dos para facilitar la auditabilidad del código, ya que en la primera parte se realizan todas las comprobaciones y en el segundo las acciones.
El array de `Units` estáordenado cronologicamente, donde la posición 0 siempre serán las `stakeUnits` que antes podrás retirar, así que la función siempre apunta a la posición 0.
Primero se comprueban las fechas. La dirección tiene que retirar todas las `fees` hasta la fecha y si las `stakeUnits` en la posición 0 se pueden retirar. Despues se comprueban las cantidades para saber si se han especificado `_unitsToUndoStake` mayores que 0 y si es mayor de la cantidad que tiene la dirección en stake. 
En la parte de la acción, se realiza un bucle donde se sigue el siguiente comportamiento cada iteración:
- Si la posición 0 se puede retirar, comprobando la fecha (`dateToUnstake`) y si `_rUnits`(unidades restantes) no es 0.
- Si se puede, comprueba que las `_rUnits`, son mayores que el `stakedUnits` de la posición 0.
- Si es mayor, se elimina el elemento del array, se restan las unidades al `_rUnits` y se vuelve a empezar.
- Si es menor, se resta a `stakedUnits` las `_rUnits` y se pone a 0.

En el caso de que la posición 0 no se pueda retirar o `_rUnits`sea 0, se termina el bucle y se procede a actualizar los `tokensStaked` de la dirección y se resta las unidades usadas del total `amountStakeUnits`.
```
function undoStake(
    uint256 _unitsToUndoStake
) 
    public 
{
    require(stake[msg.sender].dateLastWithdraw == dateWeek, "You need to withdraw the fees first");
    require(stake[msg.sender].stakeUnits[0].dateToUnstake <= block.timestamp, "You can't unstake yet");
        
    require(_unitsToUndoStake > 0, "Units must be more than 0.");
    uint256 _amount = _unitsToUndoStake.mul(pricePerStakeUnit);
    require(stake[msg.sender].tokensStaked <= _amount, "You are trying to unstake more than you have.");
    _undoStake(_unitsToUndoStake, msg.sender);
}


function _undoStake(
    uint256 _unitsToUndoStake, 
    address _user
) 
    internal 
{
    uint _rUnits = _unitsToUndoStake;

    for(uint i = 0; i < 40; i++){
        if(_rUnits > 0 && stake[_user].stakeUnits[0].dateToUnstake <= block.timestamp) {
            if (stake[_user].stakeUnits[0].stakedUnits <= _rUnits) {
                _rUnits = _rUnits.sub(stake[_user].stakeUnits[0].stakedUnits);
                removeStakeUnit(0, _user);
            } else { 
                stake[_user].stakeUnits[0].stakedUnits = stake[_user].stakeUnits[0].stakedUnits.sub(_unitsToUndoStake);
                _rUnits = 0;
            }
        } else {
            break;
        }
    }
    uint _usedUnits = _unitsToUndoStake.sub(_rUnits);
    stake[_user].tokensStaked = stake[_user].tokensStaked.sub(_usedUnits.mul(pricePerStakeUnit));
    amountStakeUnits = amountStakeUnits.sub(_usedUnits);
    }
```
_Argumentos_


- **_unitsToUndoStake** `uint256` Cantidad de unidades a reducir el stake.

### removeStakeUnit()
Función interna usada por el `undoStake()` para eliminar la primera entrada del array. 

```
function removeStakeUnit(
    uint _index, 
    address _user
) 
    internal 
{
    for (uint i = _index; i < stake[msg.sender].stakeUnits.length-1; i++){
        stake[_user].stakeUnits[i] = stake[_user].stakeUnits[i+1];
    }
    stake[_user].stakeUnits.length--;
}
```
_Argumentos_


- **_index** `uint256` Posición a eliminar del array.
- **_user** `address` Dirección a la cual reducir el array.

### calculateWeeklyFees()
Una vez a la semana, se puede llamar a esta función para calcular cuanto `fee` se le ha asignado **solo** en la última semana.
Primero comprueba que haya pasado una semana desde la última vez que se calculó(`dateWeek`). 
Después se añade 1 semana a `dateWeek` y se pide la cantidad de tokens `TEUR` que tiene el `TKB`. Con eso, se calcula las fees que son de la última semana usando el `balanceTEUR`, la cantidad de fees que se han retirado (`totalFeeWithdrawn`) y la suma de todas las fees hasta la fecha(`totalFeeCollected`). Una vez calculado, se divide por la cantidad de `stakeUnits` que hay en ese momento y se guarda el valor por unidad en el mapping `feesPerWeek`. También se añade las `fee` de esta semana al `totalFeeCollected`.

```
function calculateWeeklyFees() 
    public 
{
    require(block.timestamp >= dateWeek+1 weeks, "You can't do this yet.");
    dateWeek = dateWeek.add(1 weeks);
    uint256 balanceTEUR = TEURabi.balanceOf(this);
    uint256 _fee = (balanceTEUR.add(totalFeeWithdrawn)).sub(totalFeeCollected);
		
    uint256 _feePerUser = _fee.div(amountStakeUnits);

    totalFeeCollected = totalFeeCollected.add(_fee);
    feesPerWeek[dateWeek] = _feePerUser;
}
```

### withdrawFees()
Al llamar a esta función, se retirarán las fees asignadas a la dirección que llama de forma automática. Hay un límite de 40 semanas para cobrar, ya que al poner más, puede fallar por el coste elevado de gas de los bucles.
Primero se comprueba que la dirección no es 0x000..., que la dirección tenga algún `stakeUnit` y si tiene fees sin retirar.
En el bucle, se comprueba si la `dateLastWithdraw` es menor o igual que `dateWeek`. Si lo es, se calculan las fees que le corresponden en `feesPerWeek[_dateUser]` y se le suma 1 semana a `_dateUser`. Cuando la comprobación falla, se sale del bucle y se asigna la `_dateUser` a la `dateLastWithdraw`, la cual será `dateWeek` + 1 semana. 
Y por último, se le envian desde el `TEUR` la cantidad total de fees calculada por el bucle.

```
function withdrawFees() 
    public 
{
    address _user = msg.sender;
    require(_user != address(0), "Adress can't be 0x0000...");
    require(stake[_user].tokensStaked > 0, "You don't have any stake unit.");
    require(stake[_user].dateLastWithdraw <= dateWeek, "You don't have fees to withdraw.");

    uint256 _units = stake[_user].tokensStaked.div(pricePerStakeUnit.mul(decimals));
    uint256 _amountFees = 0;
    uint256 _dateUser = stake[_user].dateLastWithdraw;

    for (uint256 i = 1; i <= 40; i++) {
        if (_dateUser <= dateWeek) {
            _amountFees = (_amountFees.add(feesPerWeek[_dateUser])).mul(_units);
            _dateUser = _dateUser.add(1 weeks);
        } else {
            break;
        }
    }
    assert(_amountFees > 0);
    stake[_user].dateLastWithdraw = _dateUser;
    TEURabi.transfer(_user, _amountFees);
}
```


### getStakeUnits()
Devuelve la cantidad de `stakeUnits` que tiene una dirección. 

```
function getStakeUnits(
    address _user
) 
    public 
    view 
    returns (uint)
{
    return stake[_user].tokensStaked.div(pricePerStakeUnit);
}
```
_Argumentos_


- **_user** `uint256` Dirección de la cual saber la cantidad de units.

### freezeTransfer()
Solo el contrato de la [`DAICO`](/daico/) puede llamar a esta función. Al llamar, `stopped` se pone en true, lo que hace que se congelen las transferencias. 
```
function freezeTransfer() 
    public
{
    require(msg.sender == DAICOaddress, "Must be called by the DAICO contract.");
    stopped = true;
}
```

### unfreezeTransfer()
Solo el contrato de la [`DAICO`](/daico/) puede llamar a esta función. Al llamar, `stopped` se pone en false, lo que hace que se descongelen las transferencias. 
```
function freezeTransfer() 
    public
{
    require(msg.sender == DAICOaddress, "Must be called by the DAICO contract.");
    stopped = false;
}
```

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
Comprueba que `_to` no es la dirección 0x000... y que el `msg.sender` tiene suficiente balance, restando la cantidad que tiene en `stake`. Después realiza la transferencia y emite el evento.
```
function transfer(
    address _to, 
    uint256 _value
) 
    public 
    returns(bool) 
{
    require(!stopped, "Transfers are paused.");
    require(_to != address(0), "Address can't be 0x0000...");
    require(_value <= balances[msg.sender].sub(stake[msg.sender].tokensStaked), "Insufficient balance");

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
Devuelve la cantidad de tokens que tiene `_owner`, restando la cantidad que tiene en `stake`.
```
function balanceOf(address _owner) 
    public 
    view 
    returns(uint256) 
{
    return balances[_owner].sub(stake[_owner].tokensStaked);
}
```
_Argumentos_


- **_owner** `address` Dirección de quien devolver el balance.

### transferFrom()
Comprueba que `_to` no es la dirección 0x000..., que `_from` tenga suficiente balance, restando la cantidad que tiene en `stake` y que `_from` le haya concedido suficiente balance a `msg.sender`. Después reduce el balance de `_from`, incrementa el balance de `_to` y reduce el `allowance` de `_from` a `msg.sender`.
```
function transferFrom(
    address _from,
    address _to,
    uint256 _value
)
    public
    returns(bool) 
{
    require(!stopped, "Transfers are paused.");
    require(_to != address(0), "Adress can't be 0x0000...");
    require(_value <= balances[_from].sub(stake[msg.sender].tokensStaked), "Insufficient balance");
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
    require(_value != 0, "Value must be more than 0.");
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