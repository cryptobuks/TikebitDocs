En este caso, es el código original de Open Zeppelin con varias funciones enfocadas a crear un super usuario que lo único que puede hacer es retirar los permisos a la cuenta de _owner_.

## Variables
- **owner** `address` `public`
- **superUser** `address` `public`

## Eventos
```
event OwnershipRenounced(address indexed previousOwner);
event OwnershipTransferred(
       address indexed previousOwner,
       address indexed newOwner
);
event SuperUserTransferred(
    address indexed previousSu,
    address indexed newSu
);
```

## Modificadores
### onlyOwner()
Solo la dirección que tenga el rol de _owner_ puede llamar a la función.
```
modifier onlyOwner() {
    require(msg.sender == owner, "You are not the owner.");
    _;
}
```
### onlySu()
Solo la dirección que tenga el rol de _superUser_ puede llamar a la función.
```
modifier onlySu() {
    require(msg.sender == superUser, "You are not the super user.");
    _;
}
```


## Funciones
----
### constructor()
En el momento de la creación del contrato, se asigna el rol de _owner_ y _super usuario_ a la dirección que haya desplegado el contrato.
```
constructor() public {
    owner = msg.sender;
    superUser = msg.sender;
}
```
### transferSuperUser()
El poseedor de los permisos _superUsuario_ transfiere sus permisos a una nueva dirección. 
```
function transferSuperUser(address _newSu) public onlySu {
    require(_newSu != address(0), "Adress can't be 0x0000...");
    emit SuperUserTransferred(superUser, _newSu);
    superUser = _newSu;
}
```
_Argumentos_


- **_newSu** `address` Dirección del nuevo super usuario.

### renounceOwnership()
El _owner_ asigna sus permisos a la cuenta 0x000... dejando de ser el owner.
```
function renounceOwnership() public onlyOwner {
    emit OwnershipRenounced(owner);
    owner = address(0);
}
```

### transferOwnership()
El poseedor de los permisos _superUsuario_ cambia los permisos de _owner_ a la dirección proporcionada.
```
function transferOwnership(address _newOwner) public onlySu {
    require(_newOwner != address(0), "Adress can't be 0x0000...");
    emit OwnershipTransferred(owner, _newOwner);
    owner = _newOwner;
    }
```
_Argumentos_


- **_newOwner** `address` Dirección del nuevo owner.
