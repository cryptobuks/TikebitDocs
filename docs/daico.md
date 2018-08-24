# DAICO - ICO + DAO

El TKB implementa las funcionalidades de `ERC20Basic` y `ERC20`. La finalidad de este token es permitir el reparto de las `fee` que se generan en el [TEUR](/teur/). Para conseguir esto, se podrán generar `stakes`, que contienen `stakeUnits` las cuales representan las unidades proporcionales que recibirás respecto a la `feePool`. También se implementa un sistema de fechas donde está todo controlado para que se restrinja cada acción a determinada fecha. Los tokens usados en los stakes se restarán del balance de la dirección en funciones como `transfer()`, `transferFrom()` y las que requieran saber del balance de tokens de una dirección.


Las transferencias estarán congeladas hasta que se termine la ICO, la variable que controla esto es `stopped`. Es posible que una vez terminada la ICO se vuelva a para el contrato, pero solo en el caso de que se pida un `refund` de la DAICO, permitiendo así ir al contrato de la DAICO a retirar ETH a razón de los tokens TKB que tengas. Esta variable no es posible cambiarla de forma manual, se descongelará una vez termine la ICO de forma automática y al pedir un `refund` mediante la DAO.

## Variables
----

- **wallet** `address` `public`
- **icoDate** `uint256` `public`
- **hardCap** `uint256` `public`
- **softCap** `uint256` `public`
- **weiRaised** `uint256` `public`
- **withdrawDate** `uint256` `public`
- **dateUntillNextRequest** `uint256` `public`
- **voteRestrictionDate** `uint256` `public`
- **tokensSold** `uint256` `public`
- **refundVotes** `uint256` `public`
- **refundDAO** `bool` `public`
- **refundICO** `bool` `public`
- **icoFinalized** `bool` `public`
- **TKBabi** `TKB` `public`
- **requests** `Request[]` `public`

## Structs
----

Las peticiones de la DAO siguen la estructura `Request`, la cual guarda los datos básicos de cada petición y también un mapping donde se apuntará las direcciones que ya han votado. 
```
struct Request {
    string description;
    uint256 value;
    uint256 declineCount;
    uint256 dateFinalize;
    address recipient;
    bool complete;
    mapping(address => bool) decliners;
}
```
Cuando una dirección participa en la ICO, se le añade al mapping `contributors` y se le asigna la estructura `Contributor` para guardar futuros datos
```
struct Contributor {
    uint256 weiContributed;
    bool votedToRefund;
    bool gotRefundICO;
    bool gotRefundDAO;
}
```
## Mappings
----

Aquí se guardan los datos de las direcciones que participan en la ICO. También se usa en un futuro si se realiza un `refundDAO`, donde guardará si la dirección ha recibido el refund, sin importar si participó en la ICO.
``` 
mapping(address => Contributor) public contributors;
```

## Funciones
----
### constructor()
Al desplegar el contrato, se asignarán los valores de la fecha, hard cap, soft cap, etc.


_Este apartado no está terminado, ya que falta poner los valores y fechas reales_
```
constructor(
    address _owner, 
    address _token, 
    uint256 _date
) 
    public
{
    require(_owner != address(0), "Adress can't be 0x0000...");
    require(_token != address(0), "Adress can't be 0x0000...");

    Ownable.transferOwnership(_owner);

    wallet = _owner;
    TKBabi = TKB(_token);
    icoDate = _date;
    voteRestrictionDate = icoDate.add(17 weeks);
    dateUntillNextRequest = _date;
    hardCap = 500000000000000000000000000;
    softCap = 200000000000000000000000000;
}
```

### fallback()
Por temas de estructura, la función `fallback` solo llama a la función `buyTokens`.
```
function() 
    external 
    payable 
{
    buyTokens(msg.sender);
}
```

#### buyTokens()
Primero se comprueba que no haya terminado la ICO. Despues se llama a `_preValidatePurchase` para realizar más comprovaciones. Posteriormente se calcula la cantidad de tokens a recibir usando `_gerTokenAmount`. Si los tokens a recibir no superan el `hardCap`, se procede a llamar a `_processPurchase` para realizar la compra.
```
function buyTokens(
    address _user
)
    private
{
    require(block.timestamp <= icoDate, "Ico ended, you can't buy tokens.");

    uint256 weiAmount = msg.value;
    _preValidatePurchase(_user, weiAmount);
    uint256 _tokens = _getTokenAmount(weiAmount);

    require (tokensSold.add(_tokens) <= hardCap, "The hardCap is reached.");

    weiRaised = weiRaised.add(weiAmount);

    tokensSold = tokensSold.add(_tokens);

    _processPurchase(_user, weiAmount, _tokens);
}
```
_Argumentos_


- **_user** `address` Dirección a recibir los tokens.

#### _preValidatePurchase()
Comprueba que el usuario no es 0x000... y que la cantidad de `_weiAmount` no es 0.

```
function _preValidatePurchase(
    address _user,
    uint256 _weiAmount
)
    internal
    pure 
{
    require(_user != address(0), "Adress can't be 0x0000...");
    require(_weiAmount != 0, "Amount must be more than 0");
}
```
_Argumentos_


- **_weiAmount** `uint256` Cantidad de wei enviada.
- **_user** `address` Dirección a la cual reducir el array.

#### _getTokenAmount()
Al llamar con la cantidad de wei enviada, se calcula si se cumplen las condiciones del `if`. Lo que se comprueba es la cantidad de tokens ya comprados y se realizan descuentos dependiendo de que límite que se haya superado.

```
function _getTokenAmount(
    uint256 _weiAmount
)
    internal 
    view 
    returns (uint256)
{
    if (weiRaised < 400 ether && weiRaised.add(_weiAmount) <= 400 ether) {
        return _weiAmount.mul(13500);
    } else if (weiRaised < 2000 ether && weiRaised.add(_weiAmount) <= 2000 ether) {
        return _weiAmount.mul(13000);            
    } else if (weiRaised < 4400 ether && weiRaised.add(_weiAmount) <= 4400 ether) {
        return _weiAmount.mul(12750);
    } else {
        return _weiAmount.mul(12500);
    }
}
```
_Argumentos_


- **_weiAmount** `uint256` Cantidad de wei enviada.


_Devuelve_


- `uint256` Cantidad de tokens.

#### _processPurchase()
Se asigna `_weiAmount` a la dirección de `_user` en el mapping `contributors` y se llama al `TKB` para procesar la compra.

```
function _processPurchase(
    address _user,
    uint256 _weiAmount,
    uint256 _tokenAmount
)
    internal 
{
    contributors[_user].weiContributed = contributors[_user].weiContributed.add(_weiAmount);
    TKBabi.transfer(_user, _tokenAmount);
}
```
_Argumentos_


- **_user** `address` Usuario que recibe los tokens.
- **_weiAmount** `uint256` Cantidad de wei enviada.
- **_tokenAmount** `uint256` Cantidad de tokens a enviar.

### finalizeICO()
Una vez pase la fecha `icoDate`, se tiene que llamar a esta función para terminar la ICO. Si se ha superado el `softCap`, se descongela el `TKB` usando `unfreezeTransfer()` y se queman el sobrante de tokens. Si no se supera el `softCap`, se queman los tokens sobrantes y se activa `refundICO`.

```
function finalizeICO() 
    public 
{
    require(icoDate <= block.timestamp, "It's not possible to close the ICO yet.");
    require(!icoFinalized, "ICO already finalized");
    icoFinalized = true;
    uint256 tokenAmount = 500000000; // <------------

    if (tokensSold < softCap) {
        refundICO = true;
        TKBabi.burn(tokenAmount.sub(tokensSold));
    } else if (tokensSold >= softCap){
        TKBabi.unfreezeTransfer();
        // <------------
        TKBabi.burn(tokenAmount.sub(tokensSold));
    }
}
```

### refundICO()
En el caso de que esté activado `refundICO`, se activa esta función. Al llamar se comprueba que la dirección ha participado en la ICO mirando el mapping de `contributors`. Si se ha contribuido y no ha recibido refund ya, se transfiere el `wei` que se indique en `weiContributed`.
```
function refundICO()
    public 
{
    require(refundICO, "Refund isn't active");
    require(msg.sender != address(0), "Adress can't be 0x0000...");
    require(contributors[msg.sender].weiContributed != 0, "You didn't contribute to the ICO");
    require(!contributors[msg.sender].gotRefundICO, "Already withdrawn the refund.");
    address _user = msg.sender;

    contributors[_user].gotRefundICO = true;

    _user.transfer(contributors[_user].weiContributed);
}
```

### createRequest()
Solo el `owner` puede crear las peticiones. Se comprueba que haya terminado la ICO y que no está activado el refund. Luego se comprueba el sistema de fechas donde hay una fecha a partir de la cual se pueden hacer peticiones y a partir de ahí, se comprueba que el `dateUntillNextRequest` es menor que la fecha actual. También hay un límite de 800ETH por petición. 
Una vez es todo valido, se añaden `8 weeks` a `dateUntillNextRequest` y se crea la petición y se introduce en el array `requests`.
Sólo se podrá votar en contra de la petición en los 3 días siguientes a la creación de la petición.
```
function createRequest(
    string _description, 
    uint256  _value, 
    address _recipient
) 
    public 
    onlyOwner 
{
    require(icoFinalized, "ICO must end before calling this function.");
    require(!refundICO, "It's not possible to create request if the refund is active.");
    require(voteRestrictionDate < block.timestamp, "It's not time yet to create request.");
    require(dateUntillNextRequest <= block.timestamp, "It's not possible to create a request.");
    require(_value <= 800 ether, "The amount exceeds the limit.");
        
    dateUntillNextRequest = block.timestamp.add(8 weeks);

    requests.push(Request({
        description: _description,
        value: _value,
        dateFinalize: block.timestamp.add(3 days),
        recipient: _recipient,
        complete: false,
        declineCount: 0
    }));
}
```

_Argumentos_


- **_description** `string` Descripción de la petición.
- **_value** `uint256` Cantidad a retirar si se aprueba la petición.
- **_recipient** `address` Dirección a la que enviar los fondos.

### declineRequest()
Para que no se cumpla una petición, las direcciones que tengan stakeUnits en el `TKB`, tienen que llamar a esta función para votar en contra. 
Se comprueba que la petición indicada exista, que aún se pueda votar en dicha petición y que la dirección tenga algún `stakeUnit`. No se puede votar si esa dirección ha pedido el refund de la DAO o si ya has votado en dicha petición.
Si todo se cumple, se añade la cantidad de `stakeUnit` que tenga la dirección al contador de la petición `declineCount`.
```
function declineRequest(
    uint256 _index
) 
    public 
{
    require(_index < requests.length, "This request doesn't exist.");
    Request storage request = requests[_index];
    require(request.dateFinalize > block.timestamp, "The time is over to vote in this request.");

    uint256 stakeUnits = TKBabi.getStakeUnits(msg.sender);

    require(stakeUnits != 0, "You need a node to vote.");
    require(!contributors[msg.sender].votedToRefund, "You can't vote if you want to get a refund.");
    require(!request.decliners[msg.sender], "Already voted in this request.");

    request.decliners[msg.sender] = true;
    request.declineCount = request.declineCount.add(stakeUnits);
}
```
_Argumentos_


- `uint256` Indice de la petición a rechazar.

### finalizeRequest()
Cuando pasan los 3 días, el `owner` llama a esta función. 
Primero se marca como `complete` y si se ha rechazado la petición, se reduce `dateUntillNextRequest` a `1 weeks`. Si se aprueba, se envian los fondos de `value`.
```
function finalizeRequest(
    uint256 _index
) 
    public 
    onlyOwner
{
    require(_index < requests.length, "This request doesn't exist.");
    Request storage request = requests[_index];
    require(request.dateFinalize <= block.timestamp, "It's not possible to finalize this request.");
    require(!request.complete, "This request is already finalized.");

    uint256 amountStakeUnits = TKBabi.amountStakeUnits();

    request.complete = true;
    if(request.declineCount <= amountStakeUnits / 2 && amountStakeUnits / 2 > 500) {
        request.recipient.transfer(request.value);
    } else {
        dateUntillNextRequest = block.timestamp.add(1 weeks);
    }
}
```
_Argumentos_


- `uint256` Indice de la petición.

### askForRefund()
Las direcciones que tengan `stakeUnit` en el `TKB`, pueden llamar para que se haga un refund de la DAO, devolviendo los fondos que queden en la DAO.
Se comprueba que ha terminado la ICO, que no se haya hecho un `refundICO` y si se puede pedir el refund ya. Después se comprueba si tiene `stakeUnits` y si ya ha votado. 
Despues, se añade la cantidad de `stakeUnits` a `refundVotes` y a `amountVotedRefund` y se comprueba si hay votos suficientes para realizar el refund (+50%). Si se supera, SE ACTIVA `refundDAO` y se congelan las transferencias en el `TKB`.
```
function askForRefund() 
    public 
{
    require(icoFinalized, "The ICO must end before you ask for a refund.");
    require(!refundICO, "If refundICO is active, you can't ask for a refund.");
    require(voteRestrictionDate <= block.timestamp, "It's not time yet to ask for a refund.");

    uint256 stakeUnits = TKBabi.getStakeUnits(msg.sender);

    require(stakeUnits != 0, "You need 1 or more stake units to vote.");
    require(!contributors[msg.sender].votedToRefund, "You already voted to get a refund. If you want to undo this, call undoAskForRefund()");

    contributors[msg.sender].votedToRefund = true;
    contributors[msg.sender].amountVotedRefund = stakeUnits;
    refundVotes = refundVotes.add(stakeUnits);

    uint256 amountStakeUnits = TKBabi.amountStakeUnits();

    if (refundVotes > amountStakeUnits/2 && amountStakeUnits / 2 > 500) { // <-------------------------- 
        refundDAO = true;
        TKBabi.freezeTransfer();
    }
}
```

### undoAskForRefund()
Se comprueba que la ICO ha terminado, que el `refundICO` no está activo y que la dirección haya pedido el refund previamente. Luego se saca los votos que realizó la dirección cogiendo `amountVotedRefund`. Se comprueba que no es 0 y se pone a false `votedToRefund` y se restan los votos de `refundVotes`.
```
function undoAskForRefund() 
    public 
{
    require(icoFinalized, "The ICO must end before you ask for a refund.");
    require(!refundICO, "If refundICO is active, you can't ask for a refund.");
    require(!contributors[msg.sender].votedToRefund, "You didn't asked for a refund.");

    uint256 stakeUnits = contributors[msg.sender].amountVotedRefund;

    require(stakeUnits != 0, "You need 1 or more stake units to vote.");
    

    contributors[msg.sender].votedToRefund = false;
    refundVotes = refundVotes.sub(stakeUnits);
}
```


### withdrawRefundDAO()
Si se ha activado `refundDAO` y la dirección no ha pedido un refund previamente, se marca como que va a recibir un refund, se consigue la cantidad de `TKB` que tiene y si tiene más de 0, se calcula proporcionalmente el wei que hay que mandarle. 
```
function withdrawRefundDAO() 
    public 
{
    require(refundDAO, "Refund must be active.");
    require(!contributors[msg.sender].gotRefundDAO, "You already got a refund.");
        
    contributors[msg.sender].gotRefundDAO = true;
        
    uint256 TKBHold = TKBabi.balanceOf(msg.sender);

    require(TKBHold != 0, "You need to have some TKB to get the refund.");
        

    uint256 _amount = ((address(this).balance).div(tokensSold)).mul(TKBHold);

    msg.sender.transfer(_amount);
    }
```
