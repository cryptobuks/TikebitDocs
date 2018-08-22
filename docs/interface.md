Todos los códigos utilizados provienen del repositorio de [Open Zeppelin](https://github.com/OpenZeppelin/openzeppelin-solidity).

## ERC20Basic
Interfaz para la creación de tokens ERC20 con las funcionalidades básicas.
```solidity
contract ERC20Basic {
    function totalSupply() public view returns(uint256);
    function balanceOf(address who) public view returns(uint256);
    function transfer(address to, uint256 value) public returns(bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
}
```

## ERC20
Interfaz complementaria de ERC20Basic que implementa funcionalidades extra para los tokens ERC20.
```solidity
contract ERC20 is ERC20Basic {
    function allowance(address owner, address spender) public view returns(uint256);
    function transferFrom(address from, address to, uint256 value) public returns(bool);
    function approve(address spender, uint256 value) public returns(bool);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}
```

## SafeMath
Librería utilizada para la implementación de elementos de seguridad para las operaciónes matemáticas. Previene los overflows y lo undeflows.
```solidity
library SafeMath {
    function mul(uint256 a, uint256 b) internal pure returns(uint256 c) {
        if (a == 0) {
            return 0;
        }

        c = a * b;
        assert(c / a == b);
        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns(uint256) {
        return a / b;
    }

    function sub(uint256 a, uint256 b) internal pure returns(uint256) {
        assert(b <= a);
        return a - b;
    }

    function add(uint256 a, uint256 b) internal pure returns(uint256 c) {
        c = a + b;
        assert(c >= a);
        return c;
    }
}
```
